# E-Ink Display IoT System Documentation

Complete setup and maintenance guide for the wind turbine monitoring e-ink display system.

## Quick Start

### Install MkDocs

```powershell
# Install Python (if not already installed)
# Download from https://www.python.org/downloads/

# Install MkDocs and Material theme
pip install mkdocs
pip install mkdocs-material
```

### Serve Documentation Locally

```powershell
# Navigate to this directory
cd d:\development\stage

# Serve the docs
mkdocs serve
```

Open your browser to `http://127.0.0.1:8000`

### Build Static Site

```powershell
# Build HTML files
mkdocs build

# Output will be in 'site' directory
# Can be hosted on any web server
```

## Documentation Structure

```
docs/
├── index.md                    # Home page
├── architecture/
│   ├── overview.md             # System architecture
│   └── data-flow.md            # Data flow diagrams
├── infrastructure/
│   ├── proxmox.md              # Proxmox host setup
│   ├── vm-setup.md             # VM configuration
│   └── iotstack.md             # IOTstack installation
├── services/
│   ├── influxdb.md             # InfluxDB setup
│   ├── grafana.md              # Grafana configuration
│   ├── grafana-renderer.md     # Image renderer
│   ├── node-red.md             # Node-RED flows
│   └── portainer.md            # Container management
├── raspberry-pi/
│   ├── gadget-mode.md          # USB gadget setup
│   ├── scripts.md              # Shell scripts
│   └── ssh-setup.md            # SSH configuration
├── hardware/
│   ├── network.md              # Network topology
│   ├── poe.md                  # PoE+ setup
│   └── display.md              # E-Ink display
├── maintenance/
│   ├── regular-tasks.md        # Maintenance schedule
│   ├── troubleshooting.md      # Problem solving
│   └── backup.md               # Backup & restore
└── appendix/
    ├── configs.md              # Configuration reference
    └── references.md           # External resources
```

## Publishing Options

### Option 1: GitHub Pages (Free)

1. Create GitHub repository
2. Push documentation files
3. Enable GitHub Pages in repository settings
4. Select source: `gh-pages` branch (MkDocs will create this)

```powershell
# Deploy to GitHub Pages
mkdocs gh-deploy
```

Your docs will be at `https://yourusername.github.io/repo-name/`

### Option 2: GitBook (Free/Paid)

1. Go to [https://www.gitbook.com/](https://www.gitbook.com/)
2. Import from GitHub repository
3. GitBook will build and host automatically

### Option 3: Self-Host

1. Build static files: `mkdocs build`
2. Copy `site/` directory to web server
3. Configure web server (nginx, Apache, etc.)

### Option 4: Read the Docs (Free)

1. Go to [https://readthedocs.org/](https://readthedocs.org/)
2. Import repository
3. Configure build settings
4. Docs auto-build on push

## Features

### Search

Full-text search is built-in. Try it when serving locally.

### Navigation

- Use left sidebar for navigation
- Click headings to jump to sections
- Use browser back/forward buttons

### Code Blocks

All code blocks have syntax highlighting and copy buttons.

### Mobile Friendly

Documentation is responsive and works on tablets and phones.

## Customization

### Theme Colors

Edit `mkdocs.yml`:

```yaml
theme:
  palette:
    scheme: default    # or 'slate' for dark mode
    primary: indigo    # Change to: blue, red, green, etc.
    accent: indigo
```

### Logo & Favicon

Add to `docs/` directory:

- `logo.png` - Your logo
- `favicon.ico` - Browser icon

Reference in `mkdocs.yml`:

```yaml
theme:
  logo: logo.png
  favicon: favicon.ico
```

### Add Pages

1. Create new `.md` file in `docs/`
2. Add to `nav:` section in `mkdocs.yml`
3. MkDocs will rebuild automatically

## Maintenance

### Update Documentation

1. Edit markdown files in `docs/`
2. Preview changes: `mkdocs serve`
3. Rebuild: `mkdocs build`
4. Deploy: `mkdocs gh-deploy` (if using GitHub Pages)

### Add Images

1. Create `docs/images/` directory
2. Add images there
3. Reference in markdown:

```markdown
![Description](images/diagram.png)
```

### Add Diagrams

Use Mermaid diagrams (supported by Material theme):

````markdown
```mermaid
graph LR
    A[Start] --> B[Process]
    B --> C[End]
```
````

## Export to PDF

### Using mkdocs-pdf-export-plugin

```powershell
pip install mkdocs-pdf-export-plugin
```

Add to `mkdocs.yml`:

```yaml
plugins:
  - pdf-export
```

Build will generate PDF in `site/pdf/document.pdf`

### Manual Print to PDF

1. Open docs in browser
2. Use browser Print function
3. Select "Save as PDF"
4. Repeat for each page (or print all)

## Troubleshooting

### `mkdocs: command not found`

Ensure Python Scripts directory is in PATH:

```powershell
# Add to PATH (Windows)
$env:Path += ";C:\Users\<YourName>\AppData\Local\Programs\Python\Python311\Scripts"
```

Or reinstall:

```powershell
pip install --user mkdocs mkdocs-material
```

### Port 8000 already in use

Use different port:

```powershell
mkdocs serve -a localhost:8001
```

### Build errors

Check `mkdocs.yml` syntax (must be valid YAML)

## Support

- **MkDocs**: [https://www.mkdocs.org/](https://www.mkdocs.org/)
- **Material Theme**: [https://squidfunk.github.io/mkdocs-material/](https://squidfunk.github.io/mkdocs-material/)
- **Markdown Guide**: [https://www.markdownguide.org/](https://www.markdownguide.org/)

## License

Documentation created for internship project. Adjust license as needed for your organization.

---

**Created:** January 2026
**Last Updated:** January 2026
