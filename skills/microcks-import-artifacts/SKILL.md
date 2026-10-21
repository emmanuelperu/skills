---
name: microcks-import-artifacts
description: Import OpenAPI specifications into a running Microcks instance and auto-configure dispatchers based on vendor extensions. Provides shell script templates for importing specs, downloading microcks-cli, and setting up JSON_BODY or SCRIPT dispatchers.
compatibility: Requires Docker and a running Microcks instance (see microcks-local-server skill). Also requires jq and curl.
---

# Microcks Import Artifacts

Import OpenAPI specifications into Microcks and automatically configure dispatchers based on vendor extensions.

## When to Use This Skill

Use this skill when:

- **Importing an OpenAPI spec** into a running Microcks instance for mocking
- **Auto-configuring dispatchers** after writing a spec with `x-mocking-microcks-dispatcher` extensions
- **Setting up a repeatable import workflow** with shell scripts

## Prerequisites

- **Docker** installed and running
- **Microcks** started (use the `microcks-local-server` skill)
- **jq** installed (`brew install jq` / `apt install jq`)
- **curl** installed

## Quick Start

**IMPORTANT:** The import and dispatcher configuration MUST be done by running the `import-openapi.sh` shell script. Do NOT attempt to configure dispatchers manually via curl or the Microcks API: the script handles the full workflow (import, discriminant detection, Groovy script loading, API calls).

1. Create the `mocking/` directory structure:

```bash
mkdir -p mocking/dispatchers
```

2. Copy the template scripts into it:

```bash
cp templates/import-openapi.sh mocking/import-openapi.sh
cp templates/download-microcks-cli.sh mocking/download-microcks-cli.sh
chmod +x mocking/import-openapi.sh mocking/download-microcks-cli.sh
```

3. Run the import script (this is the only command needed to import AND configure dispatchers):

```bash
./mocking/import-openapi.sh path/to/your-api-spec.yaml
```

The script will:
- Check that Microcks is running
- Download `microcks-cli` if not already cached
- Import the spec into Microcks
- Read `x-mocking-microcks-dispatcher` vendor extensions
- Auto-configure dispatchers via the Microcks API

## Security Notice

This skill provides **shell script templates** that the user copies into their project. These scripts:

- **Download an external binary** (`microcks-cli` from GitHub releases of [microcks/microcks-cli](https://github.com/microcks/microcks-cli))
- **Run Docker containers** (`mikefarah/yq` for YAML-to-JSON conversion, from [mikefarah/yq](https://github.com/mikefarah/yq))
- **Make network calls** to the local Microcks REST API

Always inform the user and ask for confirmation before executing these scripts.

---

## Vendor Extension: `x-mocking-microcks-dispatcher`

Add this vendor extension at the **operation level** in your OpenAPI spec to indicate which dispatcher type to use:

| Value | Microcks Dispatcher | Behavior |
|---|---|---|
| `json_body` | `JSON_BODY` | Detects the discriminant field in request body examples; routes by field value |
| `script_header_error` | `SCRIPT` | Routes on the `error` HTTP header: `400`→`invalid_input`, `500`→`server_error`, default→`success` |
| `script` | `SCRIPT` | Loads an external Groovy script from `mocking/dispatchers/{METHOD}-{path-slug}.groovy` |

### Example

```yaml
paths:
  /orders:
    post:
      x-mocking-microcks-dispatcher: json_body
      # ...

  /health:
    get:
      x-mocking-microcks-dispatcher: script_header_error
      # ...

  /payments:
    post:
      x-mocking-microcks-dispatcher: script
      # ...
```

---

## Groovy File Convention

For operations using the `script` dispatcher, place Groovy files at:

```
mocking/dispatchers/{METHOD}-{path-slug}.groovy
```

The path slug is built by:
1. Removing the leading `/`
2. Replacing `/` with `-`
3. Removing `{` and `}`

**Examples:**
| Operation | Groovy File |
|---|---|
| `POST /events` | `mocking/dispatchers/POST-events.groovy` |
| `POST /v1/orders/{orderId}/payments` | `mocking/dispatchers/POST-v1-orders-orderId-payments.groovy` |
| `GET /users` | `mocking/dispatchers/GET-users.groovy` |

---

## Integration Workflow

The `import-openapi.sh` script performs a 6-step workflow that combines artifact import and dispatcher configuration in a single command. Understanding these steps is useful for debugging or building your own variant.

### Step 1: Check that Microcks is reachable

The script sends a single request to `{MICROCKS_URL}/api` with a 5 second connection timeout. If Microcks does not answer, the script exits with instructions to start it (or to set `MICROCKS_URL`) rather than failing later on an HTTP error.

```bash
curl -s -o /dev/null --connect-timeout 5 "${MICROCKS_URL}/api"
```

### Step 2: Import the OpenAPI spec

The script invokes `microcks-cli` to push the spec into Microcks as a primary artifact. Microcks parses the spec and creates a mock service for every operation, using the named examples to seed request/response pairs.

```bash
microcks-cli import "${SPEC_FILE}:true" \
  --microcksURL="${MICROCKS_URL}/api" \
  --keycloakClientId=foo \
  --keycloakClientSecret=bar
```

The Keycloak credentials are placeholders: the local server started by `microcks-local-server` runs without authentication.

For operations that already have a discriminating parameter or body, Microcks uses native dispatching (`URI_PARAMS`, `URI_ELEMENTS`, `QUERY_HEADER`). The script only needs to configure custom dispatchers for operations that opt in via `x-mocking-microcks-dispatcher`.

### Step 3: Find the imported service

The script converts the YAML spec to JSON (via `mikefarah/yq` in Docker), reads `info.title` and `info.version`, then looks up the matching service in Microcks:

```bash
curl -s "${MICROCKS_URL}/api/services?page=0&size=1000" \
  | jq -r --arg name "$API_TITLE" --arg version "$API_VERSION" \
      'map(select(.name == $name and .version == $version)) | .[0].id'
```

Matching on name and version guarantees that dispatchers are pushed to the right service, even when Microcks hosts several APIs.

### Step 4: Detect vendor extensions

The script finds every operation annotated with `x-mocking-microcks-dispatcher` using `jq`:

```bash
jq -r '.paths
  | to_entries[]
  | .key as $path
  | .value
  | to_entries[]
  | select(.value."x-mocking-microcks-dispatcher" != null)
  | "\(.key | ascii_upcase) \($path) \(.value."x-mocking-microcks-dispatcher")"'
```

This produces one `METHOD path dispatcher` line per operation to process.

### Step 5: Build the dispatcher payload

The payload depends on the dispatcher type:

| Type                  | Microcks dispatcher | `dispatcherRules` content |
|-----------------------|---------------------|---------------------------|
| `json_body`           | `JSON_BODY`         | `{"exp": "/<field>", "operator": "equals", "cases": {...}}`. The script picks the first request body field whose values differ across examples (preferring one that is unique per example), maps each value to its example name, and uses the first example as `default`. |
| `script_header_error` | `SCRIPT`            | A **static Groovy script** that reads the `error` header (`400` → `invalid_input`, `500` → `server_error`, default → `success`). Identical for every operation. |
| `script`              | `SCRIPT`            | The contents of `{METHOD}-{path-slug}.groovy`, looked up next to the script (`dispatchers/`) then in `mocking/dispatchers/` at the project root. |

### Step 6: Push the dispatcher via the Microcks API

The script calls `PUT /api/services/{serviceId}/operation` for each detected operation, with the dispatcher type and rules in the body:

```http
PUT /api/services/{serviceId}/operation?operationName={METHOD}%20{path}
Content-Type: application/json

{
  "defaultDelay": 0,
  "dispatcher": "JSON_BODY",
  "dispatcherRules": "{\"exp\":\"/status\",\"operator\":\"equals\",\"cases\":{...}}",
  "parameterConstraints": []
}
```

### Why dispatchers need names to match

Microcks correlates request and response examples **by example name**. After dispatching, the matched request example's name (e.g. `success`) is used to look up the response example with the same name. If names do not match across `parameters`, `requestBody`, and `responses`, Microcks falls back to the first available example. See the `microcks-openapi-mocking` skill for the full naming rules.

---

## Script Templates

### import-openapi.sh

The main import script. See [templates/import-openapi.sh](templates/import-openapi.sh).

**Usage:**
```bash
# Basic usage (Microcks on default port 8080)
./mocking/import-openapi.sh path/to/api-spec.yaml

# Custom Microcks URL
MICROCKS_URL=http://localhost:9080 ./mocking/import-openapi.sh path/to/api-spec.yaml
```

**What it does:**
1. Validates the spec file argument
2. Health-checks the Microcks instance
3. Downloads `microcks-cli` if not cached
4. Imports the spec via `microcks-cli`
5. Converts YAML to JSON via `docker run mikefarah/yq`
6. Finds operations with `x-mocking-microcks-dispatcher`
7. Configures dispatchers via the Microcks REST API
8. Prints the Microcks UI URL

### download-microcks-cli.sh

Downloads and caches the `microcks-cli` binary. See [templates/download-microcks-cli.sh](templates/download-microcks-cli.sh).

**Features:**
- Multi-platform: macOS (Darwin) and Linux, amd64 and arm64
- Cached in `~/.local/bin/` (shared across projects)
- Version configurable via `MICROCKS_CLI_VERSION` env var (default: `0.5.6`)

---

## Complementary Skills

- **[microcks-local-server](../microcks-local-server/)**: Start a local Microcks server with Docker Compose
  ```bash
  npx skills add emmanuelperu/microcks-skills --skill microcks-local-server
  ```

- **[microcks-openapi-mocking](../microcks-openapi-mocking/)**: Write OpenAPI examples that work with Microcks dispatchers
  ```bash
  npx skills add emmanuelperu/microcks-skills --skill microcks-openapi-mocking
  ```

---

## Related

- **Microcks Documentation:** https://microcks.io/documentation
- **Microcks CLI:** https://github.com/microcks/microcks-cli
- **mikefarah/yq:** https://github.com/mikefarah/yq
