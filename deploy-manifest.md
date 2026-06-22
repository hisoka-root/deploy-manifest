# Deploy Manifest Specification v1.0

**Status:** stable  
**Reference impl:** `deploy-manifest` (Rust)  
**Filenames:** `deploy.yaml`, `deploy.yml`, `deploy.toml`, `deploy.json`  
**License:** CC BY-SA 4.0 — see [LICENSE-SPEC](LICENSE-SPEC)

---

## 1. What This Is

DMS describes what an application needs to run: its build steps, runtime,
processes, environment, dependencies, storage, and networking. It does not
describe how a platform runs it. The platform reads the manifest and translates
it into whatever native model it uses (containers, VMs, Nomad jobs, Kubernetes
resources, etc.).

This specification intentionally leaves out:

- Container runtimes (Docker, Podman, etc.)
- Hypervisors
- Schedulers (Kubernetes, Nomad, etc.)
- CI/CD pipelines

It defines *intent*, not infrastructure.

---

## 2. Design

**Human readable.** A developer should be able to open a manifest and understand
it without reading this document.

**Runtime agnostic.** Must work for PHP, Node, Python, Go, Rust, Java, Bun,
Deno, static sites, and custom runtimes.

**Infrastructure agnostic.** Must work on VPS, containers, Kubernetes, Nomad,
bare metal, PaaS platforms.

**Deterministic.** The same manifest produces the same deployment regardless
of who or what interprets it.

---

## 3. Root Fields

```yaml
version: 1

app:
build:
runtime:
start:
workers:
network:
env:
services:
storage:
health:
scaling:
cron:
```

---

## 4. `version`

Required. Single integer. Current valid value is `1`. Future breaking changes
increment this number.

```yaml
version: 1
```

---

## 5. `app`

Application metadata.

```yaml
app:
  name: store
  description: Ecommerce platform
  homepage: https://example.com
  repository: https://github.com/example/store
  labels:
    team: payments
```

**Rules:**

- `name` is required
- Must be DNS-safe (lowercase alphanumeric and hyphens)
- Max 64 characters

Valid names:

```text
store
my-api
company-blog
```

Invalid:

```text
My Application!
store_frontend_v2
```

---

## 6. `runtime`

Execution environment the application expects.

```yaml
runtime:
  type: php
  version: "8.4"
```

**Supported types:** `php`, `node`, `python`, `rust`, `go`, `java`, `bun`,
`deno`, `static`, `custom`.

Custom means the platform chooses the environment. No `version` field needed:

```yaml
runtime:
  type: custom
```

---

## 7. `build`

How to turn source code into a runnable artifact.

```yaml
build:
  strategy: auto
```

**Supported strategies:** `auto`, `custom`, `dockerfile`, `buildpack`,
`nixpacks`, `none`.

Custom commands run sequentially in a shell. Non-zero exit codes fail the
deployment.

```yaml
build:
  strategy: custom
  commands:
    - composer install
    - npm install
    - npm run build
```

### Build Variables

Environment variables set only during the build phase. Same structure as
runtime `env` — plain values and secret references both work.

```yaml
build:
  strategy: custom
  commands:
    - npm run build
  variables:
    NEXT_PUBLIC_API_URL: https://api.example.com
    BUILD_SECRET:
      secret: build-env-key
```

---

## 8. `start`

Primary processes — the thing that runs when the deployment is live.

Single process:

```yaml
start:
  command: php public/index.php
```

Multiple named processes:

```yaml
start:
  web: php public/index.php
  worker: php artisan queue:work
```

**Rules:**

- At least one process required
- Process names must be unique within `start`

Conventional names:

```text
web, worker, scheduler, queue
```

These are not enforced — use them or don't. Platforms may give them special
treatment (e.g. routing HTTP to a process named `web`).

---

## 9. `workers`

Scalable background processes. Same idea as `start` but you control how many
replicas run.

```yaml
workers:
  queue:
    command: php artisan queue:work
    replicas: 3
```

**Rules:**

- `replicas` >= 1

---

## 10. `env`

Runtime environment variables.

```yaml
env:
  APP_ENV: production
```

Values can reference secrets the platform should resolve at deploy time:

```yaml
env:
  DB_PASSWORD:
    secret: database-password
```

How secrets are resolved is implementation-defined (environment injection,
mounted files, sidecars, etc.).

---

## 11. `network`

How the application is reached.

```yaml
network:
  port: 8080
  domains:
    - store.example.com
  routes:
    - /api/*
```

Port range: 1–65535.

---

## 12. `storage`

Persistent volumes attached to the application.

```yaml
storage:
  - name: uploads
    mount: /app/uploads
    size: 20GB
```

Volumes can be ephemeral (not persisted across deploys):

```yaml
storage:
  - name: cache
    mount: /cache
    ephemeral: true
```

---

## 13. `services`

Infrastructure dependencies the application expects to be available.

```yaml
services:
  database:
    type: postgres
    version: "17"
  redis:
    type: redis
    version: "7"
```

Standard service types:

```text
postgres, mysql, mariadb, redis, mongodb, valkey
rabbitmq, nats, minio, elasticsearch, opensearch
```

Implementations may support additional types.

---

## 14. `health`

How the platform checks if the application is alive.

HTTP check:

```yaml
health:
  path: /health
  interval: 30s
  timeout: 5s
```

Command check:

```yaml
health:
  command: php artisan health:check
```

Exactly one of `path` or `command` must be set. `interval` and `timeout` are
optional.

---

## 15. `scaling`

Constraints on how many instances the platform may run.

```yaml
scaling:
  min: 2
  max: 10
  cpu_target: 80
```

**Rules:**

- `min` <= `max`
- `cpu_target` is 1–100 (if set)

---

## 16. `cron`

Scheduled tasks. POSIX cron syntax.

```yaml
cron:
  - name: cleanup
    schedule: "0 0 * * *"
    command: php artisan cleanup
```

---

## 17. Validation

A valid manifest must:

- Include `version`
- Include `app.name`
- Include `runtime.type`
- Include at least one `start` command
- Pass schema validation (no unknown top-level fields other than `x-*`)

A valid implementation must:

- Reject invalid schemas
- Reject unknown versions
- Preserve unknown `x-*` fields

---

## 18. Extension System

Vendors can add custom fields under an `x-` prefix. These are ignored by other
implementations but preserved through parse–serialize round trips.

```yaml
x-chainform:
  region: us-east

x-fluxgate:
  proxy: true
```

Rules:

- Must begin with `x-`
- Must not conflict with existing fields
- Other implementations must preserve them

---

# Reference Implementation: `deploy-manifest` (Rust)

## Crate Layout

```text
deploy-manifest/
├── Cargo.toml
├── README.md
├── src/
│   ├── lib.rs
│   ├── model.rs
│   ├── error.rs
│   ├── validation.rs
│   ├── parser.rs
│   ├── yaml.rs
│   ├── json.rs
│   └── toml.rs
├── tests/
└── examples/
    ├── deploy.yaml
    ├── deploy.json
    └── deploy.toml
```

## Usage

```rust
use deploy_manifest::DeployManifest;

let manifest = DeployManifest::from_file("deploy.yaml")?;
manifest.validate()?;
```

## Parsing

```rust
// Auto-detect from extension
DeployManifest::from_file("deploy.yaml")?;

// Explicit format
DeployManifest::from_reader(reader, Format::Yaml)?;

// Format-specific
DeployManifest::from_yaml("deploy.yaml")?;
DeployManifest::from_json("deploy.json")?;
DeployManifest::from_toml("deploy.toml")?;
```

## Serialization

```rust
manifest.to_yaml()?;
manifest.to_json()?;
manifest.to_toml()?;
```

## Validation

```rust
pub trait Validate {
    fn validate(&self) -> Result<(), ValidationError>;
}
```

Returns `ValidationError` on any violation of the rules above.
