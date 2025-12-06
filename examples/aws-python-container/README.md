# ❍ Python Example with Containers

Deploy python applications using sst.

SST uses [uv](https://github.com/astral-sh/uv) to manage your python runtime. If you do not have uv installed, you can install it [here](https://docs.astral.sh/uv/getting-started/installation/). Any sst workspace package can be built and deployed to aws lambda using sst. In this example we deploy an API handler to lambda from the `functions` directory. The handler depends on shared code from the `shared` directory using uv's workspaces feature. (Note: builds currently do not tree shake so lots of workspaces can make larger builds than necessary.) We also deploy another function from `custom-dockerfile` to show how you can use a custom Dockerfile to deploy your python code.

Python functions can be deployed just like other SST functions, the only difference is that the functions themselves must be configured within a uv workspace, there is no drop-in-mode.

```typescript title="sst.config.ts"
const python = new sst.aws.Function("MyPythonFunction", {
  python: { container: true },
  handler: "functions/src/functions/api.handler",
  runtime: "python3.11",
  url: true
});
```

If you are using live lambdas for your python functions, it is recommended to specify your python version to match your Lambda runtime otherwise you may encounter issues with dependencies.

```toml title="src/pyproject.toml"
[project]
name = "aws-python"
version = "0.1.0"
description = "A SST app"
authors = [
    {name = "<your_name_here>", email = "<your_email_here>" },
]
requires-python = "==3.11.*"
```

## Docker Build Options

SST supports several Docker build options for Python container functions, powered by `@pulumi/docker-build`.

### Build Args

You can pass build arguments to the Docker build using `buildArgs`. This is useful for passing configuration values needed during the Docker build that can be baked into the image.

```typescript title="sst.config.ts"
const withBuildArgs = new sst.aws.Function("MyPythonFunction", {
  python: {
    container: true,
    buildArgs: {
      MY_BUILD_ARG: "my-value",
    },
  },
  handler: "build_args/src/build_args/api.handler",
  runtime: "python3.11",
  url: true
});
```

In your Dockerfile, declare the ARG after the FROM statement to make it available in the build stage:

```dockerfile title="build_args/Dockerfile"
ARG PYTHON_VERSION=3.11
FROM public.ecr.aws/lambda/python:${PYTHON_VERSION}

# Declare build args AFTER FROM to make them available
ARG MY_BUILD_ARG="default_value"

# Set as ENV so it's available at runtime
ENV MY_BUILD_ARG=${MY_BUILD_ARG}
```

### Secrets (for sensitive values)

For sensitive values like authentication tokens, use `secrets` instead of `buildArgs`. Secrets are mounted during the build but **not persisted in the final image**, making them ideal for:

- AWS CodeArtifact authentication tokens
- Private PyPI registry credentials
- SSH keys for private Git repositories

```typescript title="sst.config.ts"
const withSecrets = new sst.aws.Function("MyPythonFunction", {
  python: {
    container: true,
    secrets: {
      // The value is mounted at /run/secrets/<key> during build
      CODEARTIFACT_AUTH_TOKEN: process.env.CODEARTIFACT_AUTH_TOKEN,
    },
  },
  handler: "functions/src/functions/api.handler",
  runtime: "python3.11",
  url: true
});
```

In your Dockerfile, mount the secret using `RUN --mount=type=secret`:

```dockerfile
# Install private packages using secrets (not persisted in image)
RUN --mount=type=secret,id=CODEARTIFACT_AUTH_TOKEN \
    CODEARTIFACT_AUTH_TOKEN=$(cat /run/secrets/CODEARTIFACT_AUTH_TOKEN) && \
    pip install --extra-index-url \
    https://aws:${CODEARTIFACT_AUTH_TOKEN}@my-domain.d.codeartifact.us-east-1.amazonaws.com/pypi/my-repo/simple/ \
    my-private-package
```

### Multi-stage Build Targets

Use `target` to stop at a specific stage in a multi-stage Dockerfile:

```typescript
const withTarget = new sst.aws.Function("MyPythonFunction", {
  python: {
    container: true,
    target: "production",
  },
  handler: "functions/src/functions/api.handler",
  runtime: "python3.11",
  url: true
});
```

### Network Mode

Set the network mode for `RUN` instructions during the build:

```typescript
const withNetwork = new sst.aws.Function("MyPythonFunction", {
  python: {
    container: true,
    network: "host", // "default" | "host" | "none"
  },
  handler: "functions/src/functions/api.handler",
  runtime: "python3.11",
  url: true
});
```

### SSH Forwarding

Mount SSH keys for accessing private Git repositories during the build:

```typescript
const withSSH = new sst.aws.Function("MyPythonFunction", {
  python: {
    container: true,
    ssh: {
      default: ["$SSH_AUTH_SOCK"],
    },
  },
  handler: "functions/src/functions/api.handler",
  runtime: "python3.11",
  url: true
});
```

Live lambda will locally run your python code by building the workspace and running the specified handler. You can have multiple handlers in the same workspace and have multiple workspaces in the same project.

```markdown
.
├── workspace_a
│   ├── pyproject.toml
│   └── src
│       └── workspace_a
│           ├── __init__.py
│           ├── api_a.py
│           └── api_b.py
└── workspace_b
    ├── pyproject.toml
    └── src
        └── workspace_b
            ├── __init__.py
            └── index.py
```