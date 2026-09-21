# Air-Gapped Deployment Runbook

Deploy Open Notebook on a host with no internet access, using self-hosted models served by vLLM.

The approach: build and prove the whole stack on an internet-connected staging VM, freeze it into a transfer bundle, carry the bundle across, and replay it on the offline host. Nothing is downloaded on the offline side.

This guide assumes RHEL on both machines. For Debian or Ubuntu, substitute `apt-get` for `dnf` and `.deb` for `.rpm`; the SELinux and firewalld sections do not apply.

## What you are building

Four services run on the offline host. Two you transfer in, two you already have.

| Service | Port | Source |
| --- | --- | --- |
| Open Notebook (UI + API + worker) | 8502 UI, 5055 API | Transferred image |
| SurrealDB | 8000 | Transferred image |
| vLLM — chat model | your choice | Already running |
| vLLM — embedding model | your choice | You add this |

The Open Notebook image runs three processes under supervisord: the FastAPI backend, the Next.js frontend, and the background worker. The worker is not optional — source processing and embedding are async jobs that queue forever without it. Because it is inside the image, you get it for free.

Both vLLM endpoints connect through Open Notebook's `openai_compatible` provider, which is registered for all four modalities. Your chat model and your embedding model are the same integration path, just different base URLs.

The only model Open Notebook strictly requires beyond chat is an embedding model. The API enforces exactly two defaults that can never be cleared: `default_chat_model` and `default_embedding_model`. Everything else falls back to the chat model or is skipped.

## Prerequisites

The staging VM must match the offline host's CPU architecture and RHEL major version. Container images are architecture-specific and RPMs are built per major release, so a RHEL 9 staging VM for a RHEL 9 target is the pairing to aim for. RHEL 8 packages will not install cleanly on RHEL 9.

**Staging VM**: an attached RHEL subscription so `dnf` can reach the repositories, internet access, Docker, roughly 30 GB free disk, and `python3` with `pip` for the Hugging Face download tool.

**Offline host**: 8 GB RAM minimum for the Open Notebook stack itself, plus whatever your vLLM models need. Budget 20 GB of disk for the application, its database, and uploaded sources. Root or sudo access to install Docker.

**Already in place on the offline host**: a running vLLM server exposing an OpenAI-compatible chat endpoint. You need its base URL and API key (vLLM accepts any non-empty string when started without `--api-key`; Open Notebook requires at least one of base URL or key to be set).

### Decisions to make now

Pick your embedding model before you start, because it is effectively permanent. Embeddings are stored as `array<float>` with cosine similarity computed at query time, so nothing stops you swapping models later — the stored vectors simply stop being comparable. Changing it means re-embedding every source.

`BAAI/bge-large-en-v1.5` (1024 dimensions, ~1.3 GB) is a solid default. `intfloat/e5-large-v2` is an equivalent alternative. If disk or VRAM is tight, `BAAI/bge-small-en-v1.5` is ~130 MB and noticeably weaker.

Decide whether you need OCR for scanned documents. That means building a derived image with Docling baked in (step 3b), which adds 2–4 GB to the image and a further 1–2 GB of model cache, and means you maintain that image across upgrades. Without it, scanned PDFs and image files fail on upload.

Decide whether you need podcasts or audio transcription. Both need text-to-speech and speech-to-text, which vLLM does not serve. That means adding a Speaches container and its model weights to the bundle. If you skip it, everything else still works — those default slots stay empty and the features are simply unavailable.

## Phase A, step 1: Prepare the staging VM

Stand up a RHEL VM on the same major version as your offline host, with a subscription attached so `dnf` can reach the repositories.

RHEL ships Podman rather than Docker, and the two conflict. Add the Docker CE repository and let `dnf` remove the conflicting packages.

```bash
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo

sudo dnf install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

If `dnf` reports a conflict with `podman`, `buildah` or `runc`, remove them first. On RHEL 9 the usual culprit is `runc` colliding with `containerd.io`.

```bash
sudo dnf remove -y podman buildah runc
```

Start the daemon and add yourself to the `docker` group, then log out and back in.

```bash
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```

Create the bundle directory that everything will accumulate into.

```bash
mkdir -p ~/onb-bundle/{images,docker-packages,models,config}
cd ~/onb-bundle
```

## Phase A, step 2: Collect RPMs for the offline host

The offline host has no package repository, so download the RPMs and every dependency now. `dnf download --resolve` fetches the full dependency closure without installing anything.

```bash
cd ~/onb-bundle/docker-packages

dnf download --resolve --alldeps --destdir=. \
  docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

If `--alldeps` is unavailable on your RHEL version, drop it — `--resolve` alone still pulls the dependencies that are not already installed on the staging VM. That is the catch: anything already present on the staging VM is skipped, so a leaner offline host will be missing it.

To avoid that, resolve against the offline host's actual package set. Take an inventory there first and carry it to the staging VM.

```bash
# On the offline host
rpm -qa | sort > installed-offline.txt
```

Then compare against what the staging VM has, and manually download anything the offline host lacks.

```bash
# On the staging VM
rpm -qa | sort > installed-staging.txt
comm -13 installed-offline.txt installed-staging.txt
```

Check what you collected. Expect `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-compose-plugin`, `docker-buildx-plugin`, and dependencies such as `container-selinux`, `fuse-overlayfs`, `slirp4netns` and `libcgroup`.

```bash
ls -la ~/onb-bundle/docker-packages/
```

`container-selinux` matters on RHEL. Without it the Docker daemon will not start correctly under SELinux, and it is easy to miss because the staging VM usually already has it.

If your offline host already runs Docker, skip this step and verify the version instead. Compose must be v2, invoked as `docker compose`.

```bash
docker --version && docker compose version
```

## Phase A, step 3: Pull and save the container images

Pin an explicit version rather than `v1-latest`. A floating tag means your staging VM and offline host can silently diverge, and you lose the ability to reproduce the bundle.

```bash
cd ~/onb-bundle/images

export ONB_TAG=1.14.0

docker pull lfnovo/open_notebook:${ONB_TAG}
docker pull surrealdb/surrealdb:v2
```

Record the exact digests so you can prove the offline host is running what you tested.

```bash
docker images --digests | grep -E 'open_notebook|surrealdb' \
  | tee ~/onb-bundle/config/image-digests.txt
```

Save both images to a single archive. `docker save` writes the full layer set, so the result is self-contained.

```bash
docker save \
  lfnovo/open_notebook:${ONB_TAG} \
  surrealdb/surrealdb:v2 \
  | gzip > onb-images.tar.gz

ls -lh onb-images.tar.gz
```

Expect roughly 2–3 GB compressed. If you also want podcasts, add `ghcr.io/speaches-ai/speaches:latest-cpu` to the pull and save commands.

The image already solves the tokenizer problem for you. The Dockerfile pre-downloads tiktoken's `o200k_base` encoding at build time into `/app/tiktoken-cache`, deliberately outside `/app/data` so a volume mount cannot shadow it. Token counting works offline with no action on your part.

## Phase A, step 3b: Build the Docling image (OCR)

Skip this section if you do not need OCR, scanned PDFs or image sources.

Docling normally installs itself from PyPI on first boot, which cannot work air-gapped. The entrypoint probes the venv on every boot with `has_module docling` and skips the install when the package is already present, so baking it into the image sidesteps the network entirely. The venv lives in the image layer, not on the data volume, which is exactly why this has to be an image change rather than a cached install.

Two separate things must come across: the Python package, and Docling's ML models. The models are pulled from Hugging Face on first *use*, not at install time, so installing the package alone leaves you with a runtime that fails on the offline host.

### Build the derived image

Read the exact `content-core` version out of the base image and pin to it. The entrypoint pins for a reason: the extra's transitive dependencies have to stay compatible with what is already locked in the image.

```bash
cd ~/onb-bundle/images

# --entrypoint bypasses docker-entrypoint.sh. Without it the script's
# "[entrypoint] Starting Open Notebook." banner goes to stdout and is
# captured alongside the version, silently corrupting the pin below.
CCORE=$(docker run --rm --entrypoint /app/.venv/bin/python \
  lfnovo/open_notebook:${ONB_TAG} \
  -c "import importlib.metadata as m; print(m.version('content-core'))")

# Must print a bare version in brackets, e.g. [2.0.4]. Anything else
# (extra words, more than one line) means the capture picked up log output.
echo "content-core version: [$CCORE]"

cat > Dockerfile.docling <<EOF
FROM lfnovo/open_notebook:${ONB_TAG}
RUN /app/.venv/bin/python -m pip install --no-cache-dir \
    "content-core[docling]==${CCORE}"
EOF

docker build -f Dockerfile.docling -t open_notebook-docling:${ONB_TAG} .
```

Expect the image to grow by roughly 2–4 GB — Docling pulls torch and transformers.

Confirm the package is importable before going further.

```bash
docker run --rm open_notebook-docling:${ONB_TAG} \
  /app/.venv/bin/python -c "import docling; print('docling ok')"
```

### Seed the Docling models

The reliable way to populate the model cache is to run the real extraction path once, during the Phase A step 6 dry-run, using this image instead of the stock one. Start the stack, upload a scanned PDF through the UI, and let the worker pull whatever Docling needs.

`HF_HOME` defaults to `/app/data/.cache/huggingface`, which is inside the `notebook_data` volume. **Copy it out before the step 6 teardown deletes that directory.**

```bash
cp -r notebook_data/.cache/huggingface ~/onb-bundle/models/hf-cache
du -sh ~/onb-bundle/models/hf-cache    # expect roughly 1–2 GB
```

If the uploaded scan produced real text in the UI, the cache is complete and correct. If it came back empty or the source failed, fix it here on the networked VM rather than discovering it offline — that is the entire point of the dry-run.

From here on this image replaces the stock one. Use `open_notebook-docling:${ONB_TAG}` in place of `lfnovo/open_notebook:${ONB_TAG}` in the `docker save` command in step 3, and in the `image:` line of the compose file in step 5. Shipping the stock image with `OPEN_NOTEBOOK_ENABLE_DOCLING=true` set is the most likely way to get this wrong: the entrypoint tries a PyPI install, fails, logs a warning, and boots without OCR.

## Phase A, step 4: Download the embedding model

This is the piece you do not already have. Download the full repository — config files and the tokenizer matter as much as the weights, and vLLM will fail to load without them.

```bash
sudo dnf install -y python3-pip
python3 -m pip install -U "huggingface_hub[cli]"

cd ~/onb-bundle/models

huggingface-cli download BAAI/bge-large-en-v1.5 \
  --local-dir ./bge-large-en-v1.5 \
  --local-dir-use-symlinks False
```

`--local-dir-use-symlinks False` is essential. Without it you get a directory of symlinks pointing into the Hugging Face cache, and the copy you carry across will contain broken links instead of files.

Confirm you have real files, not links, and that the weights are present.

```bash
cd ~/onb-bundle/models/bge-large-en-v1.5
ls -lh
find . -type l    # must print nothing
du -sh .          # expect ~1.3 GB
```

You should see `config.json`, `tokenizer.json`, `tokenizer_config.json`, `special_tokens_map.json`, and either `model.safetensors` or `pytorch_model.bin`. If `safetensors` is present, vLLM will prefer it.

If you pick a model that requires accepting a licence, run `huggingface-cli login` on the staging VM first. The offline host never authenticates to anything.

If you also want podcasts, download the Speaches models the same way — `speaches-ai/Kokoro-82M-v1.0-ONNX` for text-to-speech and `Systran/faster-whisper-small` for speech-to-text — into `~/onb-bundle/models/`.

## Phase A, step 5: Write the deployment files

Write these into `~/onb-bundle/config/`. Four changes from the stock compose file matter here: `pull_policy` becomes `never`, the image tag is pinned, `extra_hosts` lets the container reach vLLM on the host, and the bind mounts carry a `:z` suffix so SELinux relabels them.

### docker-compose.yml

```yaml
services:
  surrealdb:
    image: surrealdb/surrealdb:v2
    pull_policy: never
    command: ["start", "--log", "info", "--user", "${SURREAL_USER}",
              "--pass", "${SURREAL_PASSWORD}", "rocksdb:/mydata/mydatabase.db"]
    user: root
    ports:
      - "127.0.0.1:8000:8000"
    volumes:
      - ./surreal_data:/mydata:z
    restart: always

  open_notebook:
    # Without OCR:  image: lfnovo/open_notebook:1.14.0
    # With OCR, the derived image from step 3b that has Docling baked in:
    image: open_notebook-docling:1.14.0
    pull_policy: never
    ports:
      - "8502:8502"
      - "5055:5055"
    env_file: .env
    volumes:
      - ./notebook_data:/app/data:z
    extra_hosts:
      - "host.docker.internal:host-gateway"
    depends_on:
      - surrealdb
    restart: always
```

Docling gets no service entry of its own. It is a Python library that content-core imports in-process inside the Open Notebook container — there is no daemon, no port and no separate container to run. That is why enabling OCR is an image change plus two environment variables, not a new compose service. Its model cache lives on the `notebook_data` volume you already mount, so no extra volume is needed either.

Keep SurrealDB bound to `127.0.0.1`. It runs with simple credentials and the app reaches it over the internal compose network regardless — publishing it on `0.0.0.0` would let anyone who can reach the host connect.

### .env

```bash
# Required — encrypts stored credentials. Any string; no default exists.
OPEN_NOTEBOOK_ENCRYPTION_KEY=replace-with-a-long-random-string

# Optional shared password. Unset means no login at all.
OPEN_NOTEBOOK_PASSWORD=replace-with-a-password

SURREAL_URL=ws://surrealdb:8000/rpc
SURREAL_USER=root
SURREAL_PASSWORD=replace-with-a-db-password
SURREAL_NAMESPACE=open_notebook
SURREAL_DATABASE=open_notebook

# Docling is baked into the image (step 3b), so the entrypoint finds it
# already installed and skips the PyPI install. Setting the flag is not
# strictly required — availability is probed by import, not by this flag —
# but it documents the intent and matches the hint the Settings UI shows.
OPEN_NOTEBOOK_ENABLE_DOCLING=true

# Turns any stray Hugging Face lookup into an immediate error instead of a
# long timeout. The model cache is already on the volume.
HF_HUB_OFFLINE=1

# Leave this OFF. It triggers a PyPI install plus a Chromium download at
# first boot that cannot succeed air-gapped.
# OPEN_NOTEBOOK_ENABLE_CRAWL4AI=false
```

Generate real secrets rather than typing placeholders:

```bash
openssl rand -base64 32
```

Deliberately absent are the provider environment variables. They are a deprecated fallback and new automation should not be built on them. Phase D provisions through the API instead, which also handles your two separate vLLM endpoints cleanly — something a single `OPENAI_COMPATIBLE_BASE_URL` cannot do.

## Phase A, step 6: Dry-run on the staging VM

Do not skip this. Debugging a broken bundle on the offline side, with no ability to pull a missing layer or re-download a config file, is where air-gapped deployments go wrong.

Simulate the offline environment by cutting the VM's network after the images are loaded, or at minimum by blocking outbound traffic. Then bring the stack up exactly as the offline host will.

```bash
cd ~/onb-bundle/config
docker compose up -d
docker compose ps
```

Watch the logs until migrations finish. Schema migrations run automatically on API startup and the log is where failures surface.

```bash
docker compose logs -f open_notebook
```

Confirm the API answers and reports the database as online.

```bash
curl -s localhost:5055/api/config | head -20
```

If you can run a throwaway vLLM instance on the staging VM, serve the embedding model here and walk through Phase D once. Proving that discovery, registration, and a real embedding call all succeed is worth the extra hour.

```bash
vllm serve ~/onb-bundle/models/bge-large-en-v1.5 \
  --served-model-name bge-large-en-v1.5 \
  --task embed \
  --port 8001
```

Tear down cleanly when you are satisfied, and remove the test database so you ship an empty one.

```bash
docker compose down
rm -rf surreal_data notebook_data
```

## Phase A, step 7: Assemble and checksum the bundle

Your bundle directory should now hold four things.

| Path | Contents | Approx size |
| --- | --- | --- |
| `images/onb-images.tar.gz` | Open Notebook (Docling build) + SurrealDB | 5–7 GB |
| `docker-packages/*.rpm` | Docker engine and dependencies | ~200 MB |
| `models/bge-large-en-v1.5/` | Embedding weights and tokenizer | 1.3 GB |
| `models/hf-cache/` | Docling layout and OCR models | 1–2 GB |
| `config/` | compose, `.env`, digests, scripts | tiny |

Generate checksums before packing, so corruption in transit is detectable rather than mysterious.

```bash
cd ~/onb-bundle
find . -type f -exec sha256sum {} \; > SHA256SUMS
```

Pack the whole thing.

```bash
cd ~
tar -czf onb-airgap-bundle.tar.gz onb-bundle/
sha256sum onb-airgap-bundle.tar.gz | tee onb-airgap-bundle.sha256
```

Note the final size — typically 4–5 GB, which matters if your transfer medium has a capacity limit or your organisation's data-diode process caps file sizes.

## Phase B: Transfer to the offline host

Move `onb-airgap-bundle.tar.gz` and `onb-airgap-bundle.sha256` across by whatever route your environment sanctions — removable media, a data diode, or an approved one-way file drop. Carry the checksum file by the same route.

On the offline host, verify before unpacking. A truncated 4 GB transfer often unpacks far enough to look fine and then fails hours later inside a container layer.

```bash
sha256sum -c onb-airgap-bundle.sha256
```

Unpack and re-verify the inner files.

```bash
tar -xzf onb-airgap-bundle.tar.gz
cd onb-bundle
sha256sum -c SHA256SUMS
```

Both checks must pass before you continue. If either fails, re-transfer rather than proceeding — there is no way to repair a partial layer without network access.

## Phase C, step 1: Install Docker on the offline host

Skip this if Docker is already present with Compose v2.

Remove Podman and its companions first — they conflict with Docker CE and the install will fail part-way otherwise.

```bash
sudo dnf remove -y podman buildah runc
```

Install every RPM in one transaction so `dnf` resolves them against each other rather than in sequence. `--disablerepo=*` stops it reaching for a repository that is not there.

```bash
cd ~/onb-bundle/docker-packages
sudo dnf install -y --disablerepo=* ./*.rpm
```

If `dnf` refuses over a missing dependency, note the exact package name. Fetch it on the staging VM and transfer it rather than forcing the install with `--nodeps`, which produces a Docker daemon that starts and then fails in ways that are hard to diagnose.

Start and enable the daemon.

```bash
sudo systemctl enable --now docker
sudo systemctl status docker --no-pager
```

Add yourself to the `docker` group, then log out and back in.

```bash
sudo usermod -aG docker $USER
```

Verify both the engine and the Compose plugin.

```bash
docker --version
docker compose version
```

## Phase C, step 2: Load images and stage the model

Load the container images from the archive.

```bash
cd ~/onb-bundle/images
docker load < onb-images.tar.gz
```

Confirm both images landed and that the digests match what you recorded on the staging VM.

```bash
docker images --digests | grep -E 'open_notebook|surrealdb'
cat ~/onb-bundle/config/image-digests.txt
```

Put the embedding model somewhere permanent, outside the bundle directory.

```bash
sudo mkdir -p /opt/models
sudo cp -r ~/onb-bundle/models/bge-large-en-v1.5 /opt/models/
sudo chown -R $USER:$USER /opt/models
```

Verify the copy survived with real files and no broken symlinks.

```bash
ls -lh /opt/models/bge-large-en-v1.5/
find /opt/models -type l    # must print nothing
```

## Phase C, step 3: Serve the embedding model

Run a second vLLM instance for embeddings, on a different port from your existing chat endpoint. Use `--task embed` and point it at the local directory — never a Hugging Face repo id, which would trigger a download attempt.

```bash
vllm serve /opt/models/bge-large-en-v1.5 \
  --served-model-name bge-large-en-v1.5 \
  --task embed \
  --host 0.0.0.0 \
  --port 8001
```

Set `HF_HUB_OFFLINE=1` in the service environment as a safety net. It turns any accidental remote lookup into an immediate clear error rather than a long timeout.

The `--served-model-name` value is the name Open Notebook will use. Keep it short and record it — Phase D needs it to match exactly.

Confirm the endpoint answers and returns a vector of the expected dimension.

```bash
curl -s localhost:8001/v1/models

curl -s localhost:8001/v1/embeddings \
  -H 'Content-Type: application/json' \
  -d '{"model":"bge-large-en-v1.5","input":"connectivity test"}' \
  | head -c 300
```

Run the same `/v1/models` check against your existing chat endpoint and note both base URLs and the model names they report. You need four values for Phase D: chat base URL, chat model name, embedding base URL, embedding model name.

Wrap both in systemd units so they survive a reboot. A stack that comes back without its models looks like an application failure and wastes debugging time.

## Phase C, step 4: Start the stack

Copy the config into place and set real secrets if you have not already.

```bash
sudo mkdir -p /opt/open-notebook
sudo cp ~/onb-bundle/config/docker-compose.yml /opt/open-notebook/
sudo cp ~/onb-bundle/config/.env /opt/open-notebook/
sudo chown -R $USER:$USER /opt/open-notebook
cd /opt/open-notebook
```

Edit `.env` and replace every placeholder. `OPEN_NOTEBOOK_ENCRYPTION_KEY` must be set before anything else — credential storage fails without it, which means Phase D cannot run.

Bring the stack up.

```bash
docker compose up -d
docker compose ps
```

Follow the logs until migrations complete and all three internal processes are running.

```bash
docker compose logs -f open_notebook
```

You are looking for migrations applying cleanly, then the API binding on 5055 and the frontend on 8502. If the log shows Docling or Crawl4AI install attempts, one of the enable flags leaked into your `.env` — remove it and recreate the container.

Confirm the API is up.

```bash
curl -s localhost:5055/health
```

### SELinux and firewalld

RHEL enforces SELinux by default. The `:z` suffix on each bind mount relabels that directory as `container_file_t` so the containers can write to it. Without it SurrealDB fails to start with permission errors that read like a disk fault.

```bash
ls -ldZ /opt/open-notebook/surreal_data /opt/open-notebook/notebook_data
```

firewalld blocks the UI port by default. Open 8502 for the users who need it, and leave 5055 closed unless something outside the host calls the API directly.

```bash
sudo firewall-cmd --permanent --add-port=8502/tcp
sudo firewall-cmd --reload
```

If Open Notebook cannot reach vLLM on the host, firewalld is the usual cause rather than anything in the application. Traffic from the container arrives over the `docker0` bridge and is treated as external. Trust the bridge interface.

```bash
sudo firewall-cmd --permanent --zone=trusted --add-interface=docker0
sudo firewall-cmd --reload
```

If a problem disappears when you run `sudo setenforce 0`, it is SELinux labelling. Put enforcement back on immediately and fix the labels rather than leaving it permissive.

## Phase D: Provision models via the API

Nothing seeds models at startup — credentials, model records, and default assignments all live in the database and are normally created by clicking through Manage → Models. This script does the same work over the API so your users never see a setup screen.

Run it once, after the stack is up. It is written to be re-runnable: creating a duplicate model returns a 400 that does not affect the rest.

```bash
#!/usr/bin/env bash
set -euo pipefail

API=http://localhost:5055/api
PASS="${OPEN_NOTEBOOK_PASSWORD:-}"
AUTH=(); [ -n "$PASS" ] && AUTH=(-H "Authorization: Bearer $PASS")

# Four values from Phase C, step 3. Use IP literals, not hostnames.
CHAT_URL="http://10.0.0.5:8000/v1"
CHAT_MODEL="your-chat-model"
EMBED_URL="http://10.0.0.5:8001/v1"
EMBED_MODEL="bge-large-en-v1.5"
VLLM_KEY="not-used-but-required"

japi() { curl -sS "${AUTH[@]}" -H 'Content-Type: application/json' "$@"; }

# 1. One credential per endpoint.
CHAT_CRED=$(japi -X POST "$API/credentials" -d "{
  \"name\": \"vLLM chat\", \"provider\": \"openai_compatible\",
  \"modalities\": [\"language\"],
  \"base_url\": \"$CHAT_URL\", \"api_key\": \"$VLLM_KEY\"
}" | jq -r .id)

EMBED_CRED=$(japi -X POST "$API/credentials" -d "{
  \"name\": \"vLLM embeddings\", \"provider\": \"openai_compatible\",
  \"modalities\": [\"embedding\"],
  \"base_url\": \"$EMBED_URL\", \"api_key\": \"$VLLM_KEY\"
}" | jq -r .id)

# 2. One model record per endpoint, linked to its credential.
CHAT_ID=$(japi -X POST "$API/models" -d "{
  \"name\": \"$CHAT_MODEL\", \"provider\": \"openai_compatible\",
  \"type\": \"language\", \"credential\": \"$CHAT_CRED\"
}" | jq -r .id)

EMBED_ID=$(japi -X POST "$API/models" -d "{
  \"name\": \"$EMBED_MODEL\", \"provider\": \"openai_compatible\",
  \"type\": \"embedding\", \"credential\": \"$EMBED_CRED\"
}" | jq -r .id)

# 3. Assign the two defaults the app cannot run without.
japi -X PUT "$API/models/defaults" -d "{
  \"default_chat_model\": \"$CHAT_ID\",
  \"default_embedding_model\": \"$EMBED_ID\"
}"

echo "chat=$CHAT_ID embedding=$EMBED_ID"
```

Use IP literals in the base URLs. Every outbound target passes through a validation helper that resolves hostnames and raises `Could not resolve hostname` when DNS fails — an IP literal returns early and skips resolution entirely. Private addresses and localhost are explicitly allowed. `host.docker.internal` also works because the compose `extra_hosts` entry puts it in the container's hosts file.

`default_chat_model` and `default_embedding_model` are the only two the API refuses to clear. The optional slots — transformation, tools, large context — fall back to the chat model when empty, so leaving them unset is correct unless you have a reason to differ.

There is an `auto-assign` endpoint that fills defaults from whatever models exist, but it only populates those same two required slots and picks by a built-in provider preference. Setting them explicitly as above is more predictable.

If you also deployed Speaches, add a third credential with modalities `["text_to_speech", "speech_to_text"]`, register those models, and extend the defaults call with `default_text_to_speech_model` and `default_speech_to_text_model`.

## Phase E: Verify the deployment

Work through these in order. Each one exercises a different layer, so the first failure tells you where to look.

- [ ] `docker compose ps` shows both services healthy
- [ ] `curl -s localhost:5055/health` returns success
- [ ] `curl -s localhost:5055/api/models/defaults` shows both required slots populated
- [ ] The UI loads at `http://<host>:8502` and prompts for the password if you set one
- [ ] Create a notebook in the UI
- [ ] Upload a text or PDF file as a source
- [ ] The source finishes processing rather than staying queued
- [ ] Ask a question in chat and get a grounded answer
- [ ] Run a search and get results ranked by relevance

With the Docling image, add two more checks. The capabilities endpoint probes what is actually importable rather than trusting the enable flag, so it is the honest answer about whether OCR is live.

```bash
curl -s localhost:5055/api/capabilities
# expect: {"docling_available": true, ...}
```

Then upload a scanned PDF and confirm it extracts real text. In Settings, the document engine can stay on `auto` — content-core routes to Docling when it is available — and the OCR toggle defaults to on. If `docling_available` is false, the image is wrong; if it is true but the scan comes back empty, the model cache did not come across.

The source-processing check is the important one. It proves the worker is alive, the embedding endpoint is reachable, and vectors are being written. If a source sits at `queued` indefinitely, check the worker first.

```bash
docker compose logs open_notebook | grep -i -E 'worker|command|embed'
```

If chat works but search returns nothing, the embedding model is the problem rather than the LLM — they are separate endpoints and fail independently.

One behaviour to expect rather than debug: the Ask feature asks the model for a JSON search strategy and parses it against a schema. Smaller or heavily quantised models sometimes return empty search terms, and the app raises a clear error naming the strategy model. If chat works but Ask consistently fails, that is a model capability issue, not a deployment fault.

## Known limitations offline

**One outbound call remains.** `GET /api/config` fetches the project's `pyproject.toml` from `raw.githubusercontent.com` to check for a newer version. It is on by default and there is no environment variable to disable it. The impact is bounded: a 10-second timeout, a 24-hour cache that also caches failures, and a warning log with no crash. Prefer a firewall rule that REJECTs rather than DROPs — a blackholed connection burns the full timeout while the UI blocks on first load, whereas a rejection fails instantly. The endpoint is also unauthenticated, so this fires even before login.

**URL and YouTube sources will not work.** Fetching remote content is inherently online. Only intranet URLs your host can actually reach will resolve. Local files, uploads and pasted text are unaffected.

**OCR and scanned PDFs work only if you built the Docling image** in step 3b and carried its model cache across. On the stock image Docling is absent, and a scanned PDF fails fast: extraction returns empty, the graph raises `ValueError`, and because `ValueError` is in the command's `stop_on` list the job is marked failed with no retry. The message reads "Could not extract any text content from this source." That is a clear failure rather than a silently empty document, but it is a failure.

**No JavaScript-rendered page capture.** That needs the Crawl4AI runtime and a bundled Chromium, also a first-boot install.

**No podcasts or audio transcription** unless you added Speaches. vLLM does not serve text-to-speech or speech-to-text.

**Single shared password, no user accounts.** There is no users table and no ownership field on any record, so everyone who logs in shares one workspace and can edit or delete anything in it. This is a documented product decision, not a gap. If colleagues need separate research, run one instance per person — an authenticating reverse proxy gives you access control at the edge but does not separate data behind it.

## Maintenance

### Changing a model endpoint

After first provisioning the database is the source of truth, not your `.env`. Editing an environment variable will not move an endpoint. Update the stored credential instead.

```bash
curl -sS -X PUT http://localhost:5055/api/credentials/<credential_id> \
  -H 'Content-Type: application/json' \
  -d '{"base_url": "http://10.0.0.9:8001/v1"}'
```

### Upgrading

Repeat Phase A with the new tag on a staging VM, transfer, `docker load`, update the image tag in `docker-compose.yml`, then `docker compose up -d`. Schema migrations run automatically on startup, so watch the logs on first boot after an upgrade.

Back up before every upgrade. Migrations are one-way in practice — rollback files exist but are applied manually.

### Backups

Two directories hold everything: `surreal_data` is the database, `notebook_data` is uploads and chat checkpoints. Stop the stack first so you copy a consistent database.

```bash
cd /opt/open-notebook
docker compose down
tar -czf onb-backup-$(date +%F).tar.gz surreal_data notebook_data
docker compose up -d
```

Store `OPEN_NOTEBOOK_ENCRYPTION_KEY` separately from the backups. Restoring a database without its key leaves every stored credential unreadable, and there is no recovery path.

### Keeping the bundle

Keep the transfer bundle and its checksums after a successful deployment. Rebuilding the offline host from a known-good bundle takes minutes; reconstructing one from scratch means another staging VM and another approved transfer.

## Related

- [Air-Gapped Deployment: Working Notes](airgapped-deployment-notes.md) — why the runbook says what it says
- [Docker Compose installation](docker-compose.md)
- [Content processing engines](../3-USER-GUIDE/content-processing-engines.md) — why Docling and Crawl4AI stay disabled
- [Security configuration](../5-CONFIGURATION/security.md)
- [Environment reference](../5-CONFIGURATION/environment-reference.md)
