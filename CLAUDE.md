# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the documentation site for KuboCD, a Kubernetes application packaging and deployment tool. The documentation is built using MkDocs with the Material theme and deployed to GitHub Pages.

KuboCD abstracts Helm chart complexity by packaging applications in OCI-compliant containers and managing deployments through declarative Releases, integrating with GitOps workflows like FluxCD.

## Common Development Commands

### Setup and Environment
```bash
# Install dependencies (first time setup)
./setup/install.sh

# Activate Python virtual environment
. ./setup/activate.sh
```

### Development Server
```bash
# Start local development server with live reload
make serve
# or manually:
. ./setup/activate.sh && mkdocs serve
```

### Publishing
```bash
# Deploy to GitHub Pages
make publish
# or manually:
. ./setup/activate.sh && mkdocs gh-deploy --clean --force
```

### Help
```bash
# Show available make targets
make help
```

## Documentation Structure

- `docs/` - Main documentation content
  - `index.md` - Homepage introducing KuboCD concepts
  - `getting-started.md` - Entry point guiding users to installation
  - `user-guide/` - Step-by-step tutorial (110-210 numbered files)
  - `reference/` - API/resource reference documentation (500-530 numbered files)
- `samples/` - Example configurations, charts, and releases
  - `charts/` - Sample Helm charts
  - `configs/`, `contexts/`, `releases/` - Example KuboCD resources
- `mkdocs.yml` - Site configuration with navigation structure
- `setup/` - Environment setup scripts and Python requirements

## Key Documentation Concepts

The user guide follows a progressive tutorial structure:
1. Installation (Kind cluster or existing cluster)
2. First deployment walkthrough
3. Advanced features (contexts, configs, ingress, cert-manager)
4. CLI usage and schema formats

The reference section documents the core KuboCD resources: Package, Release, Context, and Config.

## Navigation and Numbering

Documentation files use a numbering scheme for logical ordering:
- User guide: 110-210 (installation through advanced features)
- Reference: 500-530 (core resource types)

When adding new documentation, follow the existing numbering pattern and update `mkdocs.yml` navigation accordingly.