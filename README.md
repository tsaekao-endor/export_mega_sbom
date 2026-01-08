# Endor Labs Mega SBOM Generator

Generate a consolidated CycloneDX SBOM (Software Bill of Materials) by exporting and merging multiple project SBOMs from the Endor Labs platform. This tool automatically traverses child namespaces to find projects across your entire namespace hierarchy.

## Features

- **Multi-Project Export**: Export SBOMs for multiple projects in a single run
- **Child Namespace Traversal**: Automatically discovers and searches projects in child namespaces (up to 5 levels deep by default)
- **CycloneDX Format**: Outputs industry-standard CycloneDX JSON format
- **Component Deduplication**: Merges duplicate components across projects
- **Portfolio View**: Creates a synthetic portfolio component that links all projects
- **Namespace Distribution Report**: Shows which namespaces contained which projects

## Prerequisites

- **Python 3.7+**
- **endorctl CLI**: Installed and authenticated
  - [Install endorctl](https://docs.endorlabs.com/endorctl/install/)
  - Authenticate using `endorctl init` or via API key/token

## Installation

1. Clone this repository:
   ```bash
   git clone https://github.com/your-org/mega-sbom-export.git
   cd mega-sbom-export
   ```

2. Ensure `endorctl` is installed and authenticated:
   ```bash
   endorctl init
   ```

## Usage

### Basic Usage

1. Create a text file with your project names (one per line). Comments (lines starting with `#`) are supported:
   ```
   # GitHub projects
   https://github.com/your-org/project1.git
   https://github.com/your-org/project2.git
   
   # Azure DevOps projects
   https://dev.azure.com/your-org/project3/_git/project3
   ```

2. Run the script:
   ```bash
   python3 make_mega_sbom.py \
     -n your-namespace \
     -p projects.txt \
     -o mega-sbom.json \
     --portfolio-name "My Portfolio"
   ```

### Command Line Options

| Option | Required | Default | Description |
|--------|----------|---------|-------------|
| `-n, --namespace` | Yes | - | Parent Endor namespace (will also search child namespaces) |
| `-p, --projects-file` | Yes | - | Path to text file with project names (one per line) |
| `-o, --output` | No | `mega-sbom.cyclonedx.json` | Output SBOM file path |
| `--portfolio-name` | No | `Portfolio` | Name of the synthetic portfolio root component |
| `--max-depth` | No | `5` | Maximum depth to traverse child namespaces |
| `--no-child-namespaces` | No | `false` | Disable child namespace traversal |
| `--debug` | No | `false` | Enable debug output for namespace discovery |

### Examples

**Export with child namespace traversal (default):**
```bash
python3 make_mega_sbom.py \
  -n acme-corp \
  -p projects_acme.txt \
  -o acme-mega-sbom.json \
  --portfolio-name "ACME Corp Portfolio"
```

**Export with debug output to see namespace discovery:**
```bash
python3 make_mega_sbom.py \
  -n acme-corp \
  -p projects_acme.txt \
  -o acme-mega-sbom.json \
  --debug
```

**Export from single namespace only (no child traversal):**
```bash
python3 make_mega_sbom.py \
  -n acme-corp \
  -p projects_acme.txt \
  -o acme-mega-sbom.json \
  --no-child-namespaces
```

**Increase traversal depth for deep namespace hierarchies:**
```bash
python3 make_mega_sbom.py \
  -n acme-corp \
  -p projects_acme.txt \
  -o acme-mega-sbom.json \
  --max-depth 10
```

## How It Works

### Namespace Traversal

The script automatically discovers child namespaces using the Endor Labs API. For example, given this namespace hierarchy:

```
acme-corp
├── ado-app
│   └── acme-ado-org
│       ├── project-a
│       └── project-b
├── github-app
│   └── acme-github-org
└── gitlab
```

The script will:
1. Query `acme-corp` for direct children → finds `ado-app`, `github-app`, `gitlab`
2. Query `acme-corp.ado-app` for children → finds `acme-ado-org`
3. Query `acme-corp.ado-app.acme-ado-org` for children → finds `project-a`, `project-b`
4. Continue until max depth or no more children

### Project Resolution

For each project in your input file:
1. Search for the project UUID in the parent namespace
2. If not found, search in each discovered child namespace
3. Export the SBOM from the namespace where the project was found

### SBOM Merging

The merged SBOM:
- **Deduplicates components** by purl+version (or name+version+type if no purl)
- **Resolves bom-ref collisions** by appending unique suffixes
- **Merges dependencies** from all source SBOMs
- **Creates a portfolio root** that depends on each project's root component

## Output Files

| File | Description |
|------|-------------|
| `<output>.json` | The merged CycloneDX SBOM |
| `<output-prefix>-failed_projects.txt` | List of projects that failed to export (only created if there are failures) |

For example, if you use `-o acme-mega-sbom.json`, failures will be written to `acme-failed_projects.txt`.

## Sample Output

```
Discovering child namespaces under 'acme-corp'...
  Found child namespace: acme-corp.ado-app (depth 1)
  Found child namespace: acme-corp.github-app (depth 1)
  Found child namespace: acme-corp.ado-app.acme-ado-org (depth 2)
Found 4 namespace(s):
    - acme-corp
    - acme-corp.ado-app
    - acme-corp.github-app
    - acme-corp.ado-app.acme-ado-org
Loaded 10 project(s) from projects_acme.txt
Exporting CycloneDX SBOMs for 10 projects...
  [1/10] https://github.com/acme/project1.git
  [2/10] https://dev.azure.com/acme/project2/_git/project2
    (found in child namespace: acme-corp.ado-app.acme-ado-org)
  ...

Namespace distribution:
  acme-corp: 6 project(s)
  acme-corp.ado-app.acme-ado-org: 4 project(s)

Merging SBOMs...
Done. Wrote acme-mega-sbom.json
```

## Troubleshooting

### "ERROR unknown" or API failures

- Ensure `endorctl` is properly authenticated: `endorctl init`
- Check that you have access to the namespace and its children
- Try running with `--debug` to see detailed API responses

### Projects not found in child namespaces

- Verify the project exists in the Endor Labs UI
- Check the exact project name/URL matches what's in Endor Labs
- Use `--debug` to see which namespaces are being searched

### Permission errors on child namespaces

- Ensure your API key/token has access to the parent namespace
- Child namespace access is inherited from the parent

### "A newer version of endorctl is available"

- This warning can be ignored, but consider updating: `endorctl update`

## Scripts

| Script | Description |
|--------|-------------|
| `make_mega_sbom.py` | Main script with child namespace traversal support |
| `make_mega_sbom_old.py` | Original version that only searches within a single namespace |

## License

MIT License - See LICENSE file for details.

## Contributing

Contributions are welcome! Please submit a pull request or open an issue for any bugs or feature requests.

