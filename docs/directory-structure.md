# Directory Structure Guidelines

All applications in the daiginjo-ai organization must follow this standardized directory structure to ensure consistency and maintainability.

## Root Directory Structure

```
project-root/
├── backend/          # Backend application code
├── frontend/         # Frontend application code
├── helm/            # Helm charts for Kubernetes deployment
├── CLAUDE.md        # Claude AI configuration and project context
├── README.md        # Project documentation
├── .gitignore       # Git ignore patterns
└── docker-compose.yml # Local development environment (optional)
```

## Backend Directory Structure

The `/backend` directory should contain all server-side application code:

```
backend/
├── src/             # Source code
│   ├── controllers/ # Request handlers
│   ├── services/    # Business logic
│   ├── models/      # Data models
│   ├── middleware/  # Express/Fastify middleware
│   ├── routes/      # API route definitions
│   ├── utils/       # Utility functions
│   └── app.js       # Application entry point
├── tests/           # Test files
├── config/          # Configuration files
├── package.json     # Dependencies and scripts
├── Dockerfile       # Container definition
└── README.md        # Backend-specific documentation
```

## Frontend Directory Structure

The `/frontend` directory should contain all client-side application code:

```
frontend/
├── src/             # Source code
│   ├── components/  # React/Vue components
│   ├── pages/       # Page components
│   ├── services/    # API clients
│   ├── utils/       # Utility functions
│   ├── styles/      # CSS/SCSS files
│   └── App.js       # Application entry point
├── public/          # Static assets
├── tests/           # Test files
├── package.json     # Dependencies and scripts
├── Dockerfile       # Container definition
└── README.md        # Frontend-specific documentation
```

## Helm Directory Structure

The `/helm` directory should contain Kubernetes deployment configurations:

```
helm/
├── charts/          # Helm chart dependencies
├── templates/       # Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── configmap.yaml
├── values.yaml      # Default configuration values
├── values-dev.yaml  # Development environment values
├── values-prod.yaml # Production environment values
├── Chart.yaml       # Chart metadata
└── README.md        # Deployment documentation
```

## File Naming Conventions

- Use kebab-case for directory names: `user-service`, `api-gateway`
- Use camelCase for JavaScript/TypeScript files: `userController.js`, `apiClient.ts`
- Use kebab-case for configuration files: `docker-compose.yml`, `values-prod.yaml`
- Use UPPERCASE for environment files: `README.md`, `CLAUDE.md`, `.env`

## Required Files

Every repository must include:

- `README.md` - Project overview and setup instructions
- `CLAUDE.md` - AI assistant configuration and context
- `.gitignore` - Appropriate ignore patterns for the technology stack
- License file (if open source)

## Environment-Specific Files

- Development: `docker-compose.yml`, `.env.development`
- Production: Kubernetes manifests in `/helm` directory
- Testing: Configuration in `/tests` or test-specific directories