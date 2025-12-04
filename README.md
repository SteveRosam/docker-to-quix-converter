# Docker to Quix Conversion Prompt

Use this prompt to convert any Docker/Docker Compose project to Quix Cloud deployment format.

---

## Prompt

```
I need to convert a Docker-based project to Quix Cloud deployment format.

## What is Quix?

Quix Cloud is a platform for deploying data streaming applications. It uses a specific YAML structure for deployments.

## Quix Project Structure

A Quix project has this structure:
```
project-root/
├── quix.yaml                    # Main deployment descriptor (required)
├── service-name-1/              # One folder per service/application
│   ├── app.yaml                 # Application metadata (required)
│   ├── dockerfile               # Dockerfile (lowercase, required)
│   ├── main.py                  # Entry point (or other files)
│   └── requirements.txt         # Dependencies (if Python)
├── service-name-2/
│   ├── app.yaml
│   ├── dockerfile
│   └── ...
```

## quix.yaml Format

```yaml
# Quix Project Descriptor
# This file describes the data pipeline and configuration of resources of a Quix Project.

metadata:
  version: 1.0

# Deployments section - one entry per service
deployments:
  - name: Human Readable Name        # Display name in Quix UI
    application: folder-name         # Must match the folder name
    version: latest
    deploymentType: Service          # Usually "Service"
    resources:
      cpu: 200                       # Millicores (200 = 0.2 CPU)
      memory: 500                    # MB
      replicas: 1
    publicAccess:                    # Only if service needs external access
      enabled: true
      urlPrefix: my-service          # URL will be: https://{urlPrefix}-{project}.quix.io
    variables:                       # Environment variables
      - name: VARIABLE_NAME
        inputType: FreeText          # FreeText, Secret, or OutputTopic/InputTopic
        description: What this variable does
        required: true
        value: "default-value"       # For FreeText
      - name: SECRET_VAR
        inputType: Secret
        description: A secret value
        required: true
        secretKey: my_secret_key     # Reference to Quix secret store
      - name: output
        inputType: OutputTopic       # For Kafka topic output
        description: Output topic
        required: true
        value: topic-name

# Topics section - Kafka topics used by the pipeline
topics:
  - name: topic-name
```

## app.yaml Format

```yaml
name: folder-name                    # Must match folder name and quix.yaml application
language: python                     # Always use "python" (Quix limitation)
variables:                           # Same variables as quix.yaml but with defaultValue
  - name: VARIABLE_NAME
    inputType: FreeText
    description: What this variable does
    defaultValue: "default-value"
    required: true
  - name: SECRET_VAR
    inputType: Secret
    description: A secret value
    required: true
    secretKey: my_secret_key
dockerfile: dockerfile               # Always lowercase
runEntryPoint: main.py               # Main file to run
defaultFile: main.py                 # Default file to show in editor
```

## dockerfile Requirements

- Use lowercase filename: `dockerfile` (not `Dockerfile`)
- Expose port 80 for public services (Quix routes to port 80)
- Example for Python:

```dockerfile
FROM python:3.12-slim

ENV PYTHONUNBUFFERED=1 \
    PYTHONIOENCODING=UTF-8

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 80

ENTRYPOINT ["python3", "main.py"]
```

- Example for Node.js:

```dockerfile
FROM node:20-slim

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 80

# Set PORT=80 in quix.yaml variables
CMD ["node", "src/server.js"]
```

## Variable Types

| inputType | Use Case | Extra Fields |
|-----------|----------|--------------|
| FreeText | Regular env vars | `value` or `defaultValue` |
| Secret | Passwords, API keys | `secretKey` (reference to Quix secrets) |
| InputTopic | Kafka topic to read from | `value` (topic name) |
| OutputTopic | Kafka topic to write to | `value` (topic name) |

## Secrets Convention

For secrets, define them like this:
```yaml
# In quix.yaml
- name: MY_API_KEY
  inputType: Secret
  description: API key for service X
  required: true
  secretKey: my_api_key      # User adds secret with this key in Quix UI
```

## Conversion Steps

1. **Analyze the Docker Compose file** - Identify all services, their dependencies, ports, and environment variables

2. **For each service, create a folder** with:
   - `app.yaml` - metadata
   - `dockerfile` - copied/adapted from original (use lowercase name, expose port 80)
   - Source code files
   - Dependency files (requirements.txt, package.json, etc.)

3. **Create quix.yaml** at the root with all deployments

4. **Handle environment variables**:
   - Secrets (passwords, API keys, tokens) → `inputType: Secret`
   - Regular config → `inputType: FreeText`
   - Kafka topics → `inputType: InputTopic` or `OutputTopic`

5. **Handle networking**:
   - Services that need external access → `publicAccess.enabled: true`
   - Internal service-to-service calls → Use Quix internal networking or public URLs
   - Original ports → Change to port 80 for Quix

6. **Handle volumes**: Quix doesn't support persistent volumes the same way. Consider:
   - Using external storage (S3, GCS, etc.)
   - Using Quix state management for small data
   - Redesigning to be stateless

## Now Please Convert My Project

Here is my project structure:
[PASTE YOUR docker-compose.yml HERE]

[PASTE YOUR Dockerfile(s) HERE]

[PASTE YOUR .env or .env.example showing required environment variables]

[DESCRIBE any special requirements or dependencies]

Please:
1. Create the quix.yaml
2. Create app.yaml for each service
3. Adapt the Dockerfiles (lowercase, port 80)
4. List all secrets I need to configure in Quix
5. Note any limitations or changes needed
```

---

## Example Conversion

### Input: docker-compose.yml
```yaml
version: '3.8'
services:
  api:
    build: ./api
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgres://...
      - API_KEY=${API_KEY}
  worker:
    build: ./worker
    environment:
      - REDIS_URL=redis://redis:6379
```

### Output Structure
```
quix-project/
├── quix.yaml
├── api/
│   ├── app.yaml
│   ├── dockerfile
│   └── [source files]
└── worker/
    ├── app.yaml
    ├── dockerfile
    └── [source files]
```

### Output: quix.yaml
```yaml
metadata:
  version: 1.0

deployments:
  - name: API
    application: api
    version: latest
    deploymentType: Service
    resources:
      cpu: 500
      memory: 500
      replicas: 1
    publicAccess:
      enabled: true
      urlPrefix: api
    variables:
      - name: DATABASE_URL
        inputType: Secret
        description: PostgreSQL connection string
        required: true
        secretKey: database_url
      - name: API_KEY
        inputType: Secret
        description: API authentication key
        required: true
        secretKey: api_key
      - name: PORT
        inputType: FreeText
        description: Server port
        required: false
        value: "80"

  - name: Worker
    application: worker
    version: latest
    deploymentType: Service
    resources:
      cpu: 200
      memory: 500
      replicas: 1
    variables:
      - name: REDIS_URL
        inputType: FreeText
        description: Redis connection URL
        required: true
        value: ""
```

---

## Common Gotchas

1. **Port 80**: Quix public access always routes to port 80 inside the container
2. **Lowercase dockerfile**: Must be `dockerfile` not `Dockerfile`
3. **Language field**: Always use `python` in app.yaml regardless of actual language
4. **No persistent volumes**: Design for stateless or use external storage
5. **Service discovery**: Use public URLs or Quix internal networking, not Docker network aliases
6. **Secrets**: Never put actual secrets in YAML files, use `secretKey` references
