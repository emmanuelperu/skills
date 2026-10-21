# Script Header Error Dispatcher

The **Script Header Error Dispatcher** routes mock responses based on the `error` HTTP request header. It is the simplest way to trigger error scenarios on operations that have no discriminating parameter or body, typically `GET` without query params, or `DELETE` on a fixed path.

Internally, Microcks uses a `SCRIPT` dispatcher with a static Groovy script. The script is identical for every operation that opts in, so the import workflow can generate it automatically (no Groovy file to write).

---

## When to Use

- Operations **without path or query parameters** that differentiate examples (e.g. `GET /health`, `GET /v1/info`)
- Operations **without a request body** (`GET`, `DELETE` on fixed paths)
- Any operation where you want simple success / 400 / 500 testing via a header

## When NOT to Use

- `POST` with a request body where fields differ across examples → use [`json_body`](./json-body-dispatcher.md) instead
- Operations with path or query parameters that already differentiate examples → Microcks dispatches natively on those (no extension needed)
- Routing on more scenarios than success / 400 / 500, or on fields other than the `error` header → use [`script`](./script-dispatcher.md) instead

---

## Vendor Extension

Add `x-mocking-microcks-dispatcher: script_header_error` at the **operation level**:

```yaml
paths:
  /v1/info:
    get:
      operationId: getInfo
      x-mocking-microcks-dispatcher: script_header_error
      responses:
        '200':
          description: API info
          content:
            application/json:
              examples:
                success:
                  value: { name: "Pet Store API", version: "1.0.0" }
        '400':
          description: Bad request
          content:
            application/json:
              examples:
                invalid_input:
                  value: { error: "Bad request" }
        '500':
          description: Server error
          content:
            application/json:
              examples:
                server_error:
                  value: { error: "Internal server error" }
```

The import workflow detects this extension and pushes a Microcks `SCRIPT` dispatcher with the Groovy script below.

---

## How It Works

The generated Groovy script inspects the `error` HTTP request header and returns the corresponding example name:

| Header `error` value | Returned example name | HTTP status |
|----------------------|------------------------|-------------|
| `400`                | `invalid_input`        | 400         |
| `500`                | `server_error`         | 500         |
| _(absent or other)_  | `success`              | 2xx         |

### Generated Groovy Script

```groovy
def headers = mockRequest.getRequestHeaders()
log.info("headers: " + headers)
if (headers.hasValues("error")) {
   def error = headers.get("error", "null")
   switch(error) {
      case "400":
         return "invalid_input";
      case "500":
         return "server_error";
   }
}
return "success"
```

This script is static: it does **not** depend on the operation. The import workflow pushes the same script for every operation tagged with `script_header_error`.

---

## Example Naming Convention

Response examples **MUST** use these names, otherwise the script returns a value that does not match any example and Microcks falls back to its default behavior:

| HTTP status | Example name      |
|-------------|-------------------|
| 2xx         | `success`         |
| 400         | `invalid_input`   |
| 500         | `server_error`    |

If you need additional error codes (e.g. 404, 409), use the [`script`](./script-dispatcher.md) dispatcher instead, since `script_header_error` covers only success / 400 / 500.

---

## Testing the Mock

Once the spec is imported and the dispatcher is configured, test the three scenarios with `curl`:

```bash
# Default success path (no error header)
curl 'http://localhost:8080/rest/Pet+Store+API/1.0.0/v1/info'

# 400: invalid_input
curl 'http://localhost:8080/rest/Pet+Store+API/1.0.0/v1/info' \
  -H 'error: 400'

# 500: server_error
curl 'http://localhost:8080/rest/Pet+Store+API/1.0.0/v1/info' \
  -H 'error: 500'
```

> **Mock URL pattern**: `http://localhost:8080/rest/{API_TITLE}/{API_VERSION}/{path}`: `API_TITLE` and `API_VERSION` come from the `info` block of the OpenAPI spec (URL-encoded if they contain spaces).

---

## Related

- **[json-body-dispatcher.md](./json-body-dispatcher.md)**: Route by request body field value
- **[script-dispatcher.md](./script-dispatcher.md)**: Complex routing with custom Groovy scripts
- **[dispatcher-setup.md](./dispatcher-setup.md)**: Configure dispatchers via the Microcks REST API
- **Microcks Dispatching docs**: https://microcks.io/documentation/using/admin-features/#dispatching
