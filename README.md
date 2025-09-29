# Coding Guidelines - Daiginjo AI

This repository contains the coding guidelines and standards for all repositories in the daiginjo-ai organization. These guidelines ensure consistency, maintainability, and quality across all projects.

## Repository Structure

All applications in the daiginjo-ai organization should follow this standardized directory structure:

```
project-root/
├── backend/          # Backend application code
├── frontend/         # Frontend application code
├── helm/            # Helm charts for Kubernetes deployment
├── CLAUDE.md        # Claude AI configuration and project context
└── README.md        # Project documentation
```

## Guidelines

- [Directory Structure](./docs/directory-structure.md) - Standard project layout
- [Backend Guidelines](./docs/backend-guidelines.md) - Backend development standards
- [Frontend Guidelines](./docs/frontend-guidelines.md) - Frontend development standards
- [Helm Charts](./docs/helm-guidelines.md) - Kubernetes deployment standards
- [CI/CD Guidelines](./docs/cicd-guidelines.md) - GitHub Actions workflows and deployment standards

## Usage

Reference this repository in your project's `CLAUDE.md` file to ensure AI assistants follow these guidelines when generating or modifying code:

```markdown
# Project Guidelines

This project follows the daiginjo-ai coding guidelines:
https://github.com/daiginjo-ai/coding-guidelines

Please refer to these guidelines when making changes to the codebase.
```

## Contributing

These guidelines are living documents. Please submit pull requests for improvements or updates.
