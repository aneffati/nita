# NITA Wiki — Installation Guide

The NITA Wiki is a product documentation site built with [MkDocs](https://www.mkdocs.org/) and the [Material for MkDocs](https://squidfun.github.io/mkdocs-material/) theme. It includes Mermaid diagrams, dark mode, and a fully structured technical reference.

---

## Prerequisites

- **Operating System:** Ubuntu 22.04 LTS (or similar)
- **Python:** 3.8 or later
- **pip:** Python package manager
- **Network:** Port 8000 open for serving the wiki

---

## Installation Steps

### 1. Update System Packages

```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Install Python & pip

```bash
sudo apt install -y python3 python3-pip python3-venv
```

### 3. Install MkDocs and Extensions

```bash
pip3 install mkdocs mkdocs-material mkdocs-minify-plugin
```

This installs:

| Package | Purpose |
|---|---|
| `mkdocs` | Static site generator for documentation |
| `mkdocs-material` | Material theme with Mermaid diagram support |
| `mkdocs-minify-plugin` | Minifies HTML output for faster loading |

### 4. Verify Installation

```bash
mkdocs --version
```

Expected output: `mkdocs, version 1.x.x`

---

## Running the Wiki

### Clone or Copy the Wiki Files

If you don't already have them, clone the NITA repository:

```bash
git clone https://github.com/Juniper/nita.git
cd nita/examples/nita-wiki
```

### Serve Locally (localhost only)

```bash
mkdocs serve
```

Access at: `http://127.0.0.1:8000`

### Serve on a Network Interface (for remote access)

```bash
mkdocs serve -a 0.0.0.0:8000
```

Access at: `http://<YOUR-VM-IP>:8000`

**Example:** If your VM IP is `192.168.0.30`:

```
http://192.168.0.30:8000
```

### Change the Port

```bash
mkdocs serve -a 0.0.0.0:9090
```

---

## Building a Static Site

To generate a static HTML site for deployment to any web server:

```bash
mkdocs build
```

This creates a `site/` directory containing the complete static website. You can serve it with Nginx, Apache, or any static file server.

**Example with Python's built-in server:**

```bash
cd site/
python3 -m http.server 8080
```

---

## Wiki Structure

```
nita-wiki/
├── mkdocs.yml              # MkDocs configuration
└── docs/
    ├── index.md            # Home / Product overview
    ├── architecture.md     # System architecture (Mermaid diagrams)
    ├── components.md       # Component breakdown
    ├── installation.md     # NITA installation guide
    ├── configuration.md    # Configuration reference
    ├── cli-reference.md    # nita-cmd CLI reference
    ├── projects.md         # Projects & workflows
    ├── examples.md         # Example projects walkthrough
    ├── kubernetes.md       # Kubernetes operations
    ├── containers.md       # Container management
    ├── certificates.md     # Certificate management
    ├── deployment.md       # Deployment guide
    ├── api.md              # API reference
    ├── troubleshooting.md  # Troubleshooting guide
    └── contributing.md     # Contributing guide
```

---

## Quick One-Liner

```bash
sudo apt update && sudo apt install -y python3 python3-pip && pip3 install mkdocs mkdocs-material mkdocs-minify-plugin && mkdocs serve -a 0.0.0.0:8000
```

---

## Troubleshooting

| Issue | Solution |
|---|---|
| `mkdocs: command not found` | Add pip bin to PATH: `export PATH=$PATH:$HOME/.local/bin` |
| `fence_mermaid` error | Ensure `mkdocs.yml` uses `fence_code_format` instead of `fence_mermaid` |
| Cannot access from browser | Use `-a 0.0.0.0:8000` and check firewall: `sudo ufw allow 8000/tcp` |
| Mermaid diagrams not rendering | Verify `mkdocs-material` is installed: `pip3 show mkdocs-material` |

---

## License

Copyright 2024, Juniper Networks, Inc. — MIT License
