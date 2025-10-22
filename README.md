# AI Stack Containers on macOS

This project uses Apple’s `container` CLI to run two services locally:

- **Ollama** – serves language models over HTTP so they can be pulled and run (`container` name: `ollama`).
- **Open WebUI** – a web front end that connects to Ollama (`container` name: `open-webui`).

Both containers share the custom network `ai-stack` and persist data on dedicated volumes so the setup survives reboots.

## Current Configuration

| Service      | Container | Image                               | Host Ports | Network IP (example) |
|--------------|-----------|-------------------------------------|------------|-----------------------|
| Ollama       | `ollama`  | `docker.io/ollama/ollama:latest`    | `11434`    | `192.168.65.5`        |
| Open WebUI   | `open-webui` | `ghcr.io/open-webui/open-webui:main` | `3210 → 8080` | `192.168.65.6`        |

IP addresses are assigned from the `ai-stack` network (subnet `192.168.65.0/24`) at startup and may change after restarts. See [Keeping WebUI Connected to Ollama](#keeping-webui-connected-to-ollama) for how to handle that.

## Volumes and Data Locations

| Volume          | Purpose                         | Host Path                                                                                                  |
|-----------------|---------------------------------|--------------------------------------------------------------------------------------------------------------|
| `ollama-data`   | Model files, Ollama cache       | `~/Library/Application Support/com.apple.container/volumes/ollama-data/volume.img`                          |
| `open-webui-data` | User settings, chats, embeddings | `~/Library/Application Support/com.apple.container/volumes/open-webui-data/volume.img`                     |

These `.img` files are ext4 disk images mounted inside the containers. Back them up to preserve models/history.

## Initial Setup Commands

```bash
# Run once to prepare the environment (downloads the Kata kernel if needed)
container system start

# Create shared network
container network create ai-stack

# Create persistent volumes
container volume create ollama-data
container volume create open-webui-data

# Pull images
container image pull ollama/ollama
container image pull ghcr.io/open-webui/open-webui:main

# Start services (current resource sizes, adjust to taste)
container run --name ollama --detach \
  -p 11434:11434 \
  -v ollama-data:/root/.ollama \
  --network ai-stack \
  --memory 8G --cpus 6 \
  ollama/ollama

container run --name open-webui --detach \
  -p 3210:8080 \
  -v open-webui-data:/app/backend/data \
  --network ai-stack \
  -e OLLAMA_BASE_URL=http://192.168.65.5:11434 \
  ghcr.io/open-webui/open-webui:main
```

> ⚠️ Update the `OLLAMA_BASE_URL` value if the Ollama container starts with a different IP (see next section).

## Keeping WebUI Connected to Ollama

Both containers share the `ai-stack` network, so Open WebUI can reference Ollama either by **IP address** or by **container hostname**. Apple’s implementation does not currently resolve container hostnames reliably, so using the IP address is safer.

1. After (re)starting Ollama, find its address:
   ```bash
   container inspect ollama | jq -r '.[0].networks[0].address'
   ```
   Example output: `192.168.65.5/24`.

2. If the address changed, recreate Open WebUI with the new value:
   ```bash
   container stop open-webui
   container rm open-webui
   container run --name open-webui --detach \
     -p 3210:8080 \
     -v open-webui-data:/app/backend/data \
     --network ai-stack \
     -e OLLAMA_BASE_URL=http://<new-ip>:11434 \
     ghcr.io/open-webui/open-webui:main
   ```

To avoid manual edits, you can also set `OLLAMA_BASE_URL` to `http://host.docker.internal:11434` and publish `11434` on Ollama; this works as long as both services run on the same host.

## Routine Use

### Starting the stack after a reboot

```bash
container system start           # required after every macOS reboot
container start ollama
container start open-webui
```

Check status:
```bash
container ls
```

### Accessing the services

- Open WebUI UI: <http://localhost:3210>
- Ollama API: <http://localhost:11434>

### Managing models with Ollama

```bash
# List installed models
container exec ollama ollama list

# Download a model (example)
container exec ollama ollama pull phi3:mini

# Run a prompt (example)
container exec ollama ollama run phi3:mini "Hello!"
```

Large models require additional memory. Adjust the `--memory` flag during `container run` if needed.

### Watching logs

```bash
container logs -f ollama
container logs -f open-webui
```

### Stopping and removing

```bash
container stop open-webui
container stop ollama
container rm open-webui
container rm ollama
```

Volumes remain untouched until removed explicitly:

```bash
container volume rm open-webui-data
container volume rm ollama-data
```

## Troubleshooting

- **“localhost refused to connect”** – ensure `container ls` shows both containers running; check logs for startup downloads.
- **“memory layout cannot be allocated”** – either pick a smaller model or increase Ollama’s memory (`--memory`, `--cpus`) and re-run the container.
- **Open WebUI cannot reach Ollama** – verify `OLLAMA_BASE_URL`; test from inside the WebUI container:
  ```bash
  container exec open-webui python -c "import urllib.request; print(urllib.request.urlopen('http://<ollama-ip>:11434/api/version').read())"
  ```
- **Port already in use** – change host ports (`-p host:container`) or stop the conflicting service.

## Useful References

- `container --help` – top-level CLI help.
- `container system status` – verifies the Apple container service.
- Ollama docs: <https://ollama.com>
- Open WebUI project: <https://github.com/open-webui/open-webui>

---

With the volumes and network in place, the stack is repeatable: rerun the two `container run` commands (with updated IP if needed) to get back to the same state.
