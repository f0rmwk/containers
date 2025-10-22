# Agent Handoff Notes

## Current Environment

- Repository path: `/Users/form/containers`
- Git remote: `git@github.com:f0rmwk/containers.git`
- `container` CLI (Apple’s virtualization framework) is installed and initialized with the bundled Kata kernel.

## Running Services

Two containers make up the AI stack:

1. **Ollama**
   - Name: `ollama`
   - Image: `docker.io/ollama/ollama:latest`
   - Custom network: `ai-stack`
   - Resources: `--memory 8G --cpus 6`
   - Host port: `11434`
   - Persistent volume: `ollama-data` → `~/Library/Application Support/com.apple.container/volumes/ollama-data/volume.img`
   - Purpose: serves models and responds to Open WebUI.

2. **Open WebUI**
   - Name: `open-webui`
   - Image: `ghcr.io/open-webui/open-webui:main`
   - Custom network: `ai-stack`
   - Host port mapping: `3210:8080`
   - Environment: `OLLAMA_BASE_URL=http://192.168.65.5:11434` (update if Ollama IP changes)
   - Persistent volume: `open-webui-data` → `~/Library/Application Support/com.apple.container/volumes/open-webui-data/volume.img`
   - Purpose: web UI that connects to the Ollama backend.

The network `ai-stack` spans `192.168.65.0/24`. Container IPs can shift each time they start; see README for how to refresh `OLLAMA_BASE_URL`.

## Files to Review

- `README.md` – full setup and troubleshooting guide.
- `agents.md` (this file) – quick status for incoming agents.

## Common Tasks

- **Start stack after reboot**
  ```bash
  container system start
  container start ollama
  container start open-webui
  ```

- **Check IPs**
  ```bash
  container inspect ollama | jq -r '.[0].networks[0].address'
  ```
  Recreate `open-webui` with updated `OLLAMA_BASE_URL` if needed.

- **Model management**
  ```bash
  container exec ollama ollama pull <model>
  container exec ollama ollama run <model> "prompt"
  ```

- **Logs**
  ```bash
  container logs -f ollama
  container logs -f open-webui
  ```

## Pending / Future Considerations

- Automate startup (launchd script) if auto-boot is desired.
- Consider using a hostname-based `OLLAMA_BASE_URL` once Apple’s container DNS is reliable.
- If memory errors return, increase the `--memory` value when recreating `ollama`.

Everything currently runs with local persistence and no additional services.
