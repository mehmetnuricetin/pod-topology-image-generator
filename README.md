# kubectl pod-topology

A kubectl plugin that analyzes your Kubernetes cluster and generates a visual traffic flow diagram showing the full path from Ingress to Service to Pod, including live pod health status.

```
Internet --> Ingress --> Service --> Pod (Running / Pending / Failed)
```

The diagram can be rendered as a PNG image using Luma AI's UNI-1 model or OpenAI's DALL-E 3, or generated as a text/HTML flowchart using Mermaid.js.

---

## How it works

1. **Gather** - Calls `kubectl get ingresses`, `kubectl get services`, and `kubectl get pods` against your live cluster.
2. **Analyze** - Sends the resource data to OpenAI (`gpt-4o`) which maps Ingress rules to backend Services and Services to their selected Pods via label selectors, producing a structured traffic flow summary.
3. **Generate** - Injects the summary into a visual-style prompt and submits it to the selected provider (Luma AI UNI-1, OpenAI DALL-E 3), or generates an interactive Mermaid.js diagram directly. If `LUMA_API_KEY` is not present, it will automatically fall back to OpenAI DALL-E 3.

---

## Prerequisites

- Python 3.9 or newer
- `kubectl` installed and configured with a valid kubeconfig
- An OpenAI API key (for cluster analysis and OpenAI/Mermaid rendering)
- A Luma AI API key (optional, only needed if using Luma AI generation)

---

## Installation

```bash
# Clone or download the repo
cd pod-topology-image-generator

# Run the installer (installs the plugin + Python dependencies)
./install.sh
```

The installer:
- Copies `kubectl-pod_topology` to `/usr/local/bin` (or `~/bin` if `/usr/local/bin` is not writable)
- Runs `pip install -r requirements.txt`
- Creates a `.env` file from `.env.example` if one does not already exist

---

## Configuration

Copy `.env.example` to `.env.local` and fill in your API keys:

```bash
cp .env.example .env.local
```

`.env.local`:
```
OPENAI_API_KEY=sk-...
LUMA_API_KEY=luma-... (optional, only required for luma provider)
```

The `.env.local` and `.env` files are listed in `.gitignore` and will never be committed to version control.

---

## Usage

```bash
# Generate topology for one pod in a namespace
kubectl pod-topology <pod-name> -n <namespace>

# Example with explicit output path
kubectl pod-topology nginx-topology-65b47b4648-4nktm -n default -o /tmp/staging-topology.png

# Prefix matching is supported for convenience
kubectl pod-topology nginx-topology -n default

# Generate using DALL-E 3 (OpenAI)
kubectl pod-topology nginx-topology -n default --provider openai

# Generate as a local Mermaid.js interactive HTML flowchart
kubectl pod-topology nginx-topology -n default --provider mermaid
```

### Options

| Flag | Short | Default | Description |
|------|-------|---------|-------------|
| `pod_name` | n/a | required | Target pod name (exact or prefix). |
| `--namespace` | `-n` | `default` | Namespace to inspect. Use `all` for all namespaces. |
| `--output` | `-o` | `pod-topology.png` (or `pod-topology.html` for mermaid) | Output path for the generated image/diagram. |
| `--provider` | `-p` | `luma` | Diagram generation provider: `luma` (Luma AI), `openai` / `dalle` (DALL-E 3), `mermaid` (HTML/Markdown flowchart). Falls back to `openai` if `LUMA_API_KEY` is not set. |

---

## Example output

The generated image shows:

- A globe on the left representing external internet traffic
- Ingress resources (teal) with their hostnames and TLS status
- Services (blue) with their ports
- Pods (right side) with a status badge per pod:
  - Green - Running
  - Yellow - Pending
  - Red - Failed
  - Bright red - CrashLoopBackOff
  - Grey - Unknown
- Labeled arrows showing protocol and port for each hop

---

## Project structure

```
pod-topology-image-generator/
├── kubectl-pod_topology   # main plugin script (Python, executable)
├── install.sh             # installer script
├── requirements.txt       # Python dependencies
├── .env.example           # API key template (safe to commit)
├── .env                   # your actual API keys (never committed)
└── .gitignore
```

---

## Dependencies

| Package | Purpose |
|---------|---------|
| `openai` | Cluster analysis via GPT-4o |
| `python-dotenv` | Load API keys from `.env` |
| `requests` | Download the generated image |

---

## Security notes

- API keys are loaded from local `.env.local` or `.env` files and are never hard-coded.
- `.env.local` and `.env` are in `.gitignore` to prevent accidental exposure.
- Generated images are also excluded from version control by default.
- The plugin only issues read-only `kubectl get` commands and never modifies cluster state.
