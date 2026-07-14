# SingleStore SQL Language Server

*Attention*: The code in this repository is intended for experimental use only and is not fully tested, documented, or supported by SingleStore. Visit the SingleStore Forums to ask questions about this repository.

An LSP server for SingleStore SQL providing context-aware code completion and semantic syntax highlighting. Communicates using JSON-RPC 2.0 over TCP or WebSocket protocols.

## Installation

### Linux (deb)

```bash
sudo dpkg -i s2-language-server_<version>_linux_amd64.deb
```

The binary is installed to `/usr/bin/s2-language-server` and grammar files to `/usr/share/s2-language-server/grammar_dir/`.

### Linux (rpm)

```bash
sudo rpm -i s2-language-server_<version>_linux_amd64.rpm
```

Same paths as deb: binary at `/usr/bin/s2-language-server`, grammar at `/usr/share/s2-language-server/grammar_dir/`.

### Linux / macOS (tar.gz)

```bash
tar xzf s2-language-server_<version>_<os>_<arch>.tar.gz
```

Extracts `s2-language-server` binary and `grammar_dir/` (containing the default `ddl.bin`/`dml.bin` plus any version-specific grammars such as `ddl_9.0.30.bin`) into the current directory. Place them wherever you prefer and reference `grammar_dir` with the `-grammar_dir` flag.

### Windows (zip)

Extract `s2-language-server_<version>_windows_amd64.zip`. Contains `s2-language-server.exe` and `grammar_dir\` with the grammar binary files.

The `s2-language-server.exe` binary is code-signed using [Azure Trusted Signing](https://learn.microsoft.com/en-us/azure/trusted-signing/), so Windows SmartScreen and Authenticode consumers can verify its authenticity and integrity. You can inspect the signature via **File → Properties → Digital Signatures** in Windows Explorer, or from PowerShell:

```powershell
Get-AuthenticodeSignature .\s2-language-server.exe
```

A valid signature reports `Status : Valid` and lists SingleStore as the signer.

### Docker

Prebuilt container images are published to GitHub Container Registry. Pull the image with:

```bash
docker pull ghcr.io/singlestore-labs/language-server
```

Run the language server in a container, exposing the listening port to the host:

```bash
docker run --rm -p 8080:8080 ghcr.io/singlestore-labs/language-server
```

The image ships with the `s2-language-server` binary and the grammar files preinstalled, so no additional setup is required. Flags can be passed after the image name to override defaults — for example, to run in WebSocket mode:

```bash
docker run --rm -p 8080:8080 ghcr.io/singlestore-labs/language-server \
  -mode=websocket -addr=:8080
```

#### Verifying Docker image signatures

All published container images are signed with [Sigstore Cosign](https://docs.sigstore.dev/cosign/overview/) using keyless signing backed by GitHub Actions OIDC. You can verify an image before pulling or running it to make sure it was built and published by this repository's official release workflow.

Install `cosign` (see the [installation guide](https://docs.sigstore.dev/cosign/system_config/installation/)) and then run:

```bash
cosign verify \
  --certificate-identity-regexp '^https://github\.com/singlestore/language-server/\.github/workflows/release\.yml@refs/tags/.+$' \
  --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' \
  ghcr.io/singlestore-labs/language-server:<tag>
```

Replace `<tag>` with the specific image tag you want to verify (for example a release version such as `v0.1.11`, or `latest`). To pin verification to an immutable digest instead of a mutable tag, use the `image@sha256:<digest>` form:

```bash
cosign verify \
  --certificate-identity-regexp '^https://github\.com/singlestore/language-server/\.github/workflows/release\.yml@refs/tags/.+$' \
  --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' \
  ghcr.io/singlestore-labs/language-server@sha256:<digest>
```

A successful verification prints the signature payloads and exits with status `0`. Any mismatch in the signer identity, OIDC issuer, or signature causes `cosign` to exit non-zero — do not run images that fail verification.

## Usage

```bash
s2-language-server [flags]
```

### Flags

| Flag | Default | Description |
|------|---------|-------------|
| `-mode` | `tcp` | Communication protocol: `tcp` or `websocket` |
| `-addr` | `:8080` | Address to listen on |
| `-grammar_dir` | `""` | Path to the directory containing grammar binary files (default `ddl.bin`, `dml.bin` and optional version-specific `<name>_<semver>.bin`) |
| `-max_files_with_states` | `100` | Maximum number of files to keep parser states for |
| `-max_states_per_file` | `1000` | Maximum number of parser states to keep per file |
| `-allow_all_origins` | `true` | Allow WebSocket connections from any origin (WebSocket mode only) |
| `-tls` | `false` | Enable TLS on the listener (`wss://` for `-mode=websocket`, TLS-over-TCP for `-mode=tcp`) |
| `-tls_cert` | `""` | Path to PEM-encoded TLS server certificate (required when `-tls`) |
| `-tls_key` | `""` | Path to PEM-encoded TLS server private key (required when `-tls`) |
| `-tls_min_version` | `"1.2"` | Minimum TLS version: `1.2` or `1.3` |
| `-health_addr` | `"127.0.0.1:8081"` | Address for the plain-HTTP health probe server (`/healthz`). Set to empty (`""`) to disable. See [Health Probe](#health-probe). |
| `-schema_update_interval` | `5m` | Interval for updating the schema cache (Go duration, must be > 0; e.g. `30s`, `5m`) |

### Examples

**TCP mode** (default):

```bash
s2-language-server -mode=tcp -addr=:8080 -grammar_dir=/usr/share/s2-language-server/grammar_dir
```

**WebSocket mode**:

```bash
s2-language-server -mode=websocket -addr=:8080 -grammar_dir=/usr/share/s2-language-server/grammar_dir
```

**tar.gz / zip install** (grammar directory next to the binary):

```bash
./s2-language-server -mode=tcp -addr=:3000 -grammar_dir=./grammar_dir
```

### Communication Protocols

The server supports two communication protocols:

#### TCP (default)

Direct TCP socket communication. Suitable for local connections and scenarios where a persistent TCP connection is available. This is the default mode.

```bash
s2-language-server -addr=:8080 -grammar_dir=./grammar_dir
```

Or explicitly:

```bash
s2-language-server -mode=tcp -addr=:8080 -grammar_dir=./grammar_dir
```

#### WebSocket

WebSocket-based communication over HTTP. Useful for browser-based clients, remote connections, or scenarios requiring HTTP transport. The server listens on an HTTP address and upgrades connections to WebSocket on the root path (`/`).

```bash
s2-language-server -mode=websocket -addr=:8080 -grammar_dir=./grammar_dir
```

Clients connect to `ws://<host>:8080/` (or `wss://` for secure WebSocket).

By default, the server accepts WebSocket connections from any origin. To restrict connections to same-origin only, use:

```bash
s2-language-server -mode=websocket -addr=:8080 -allow_all_origins=false -grammar_dir=./grammar_dir
```

### TLS

Both transports support TLS. Pass `-tls` together with a PEM cert and key, and the same listener will speak `wss://` (in WebSocket mode) or accept TLS-wrapped TCP (in TCP mode):

```bash
s2-language-server -mode=websocket -addr=:8443 -grammar_dir=./grammar_dir \
    -tls -tls_cert=/etc/lsp/tls.crt -tls_key=/etc/lsp/tls.key
```

```bash
s2-language-server -mode=tcp -addr=:8444 -grammar_dir=./grammar_dir \
    -tls -tls_cert=/etc/lsp/tls.crt -tls_key=/etc/lsp/tls.key
```

Clients connect to `wss://<host>:8443/` for the WebSocket transport, or use `tls.Dial` (or your language's equivalent) for the TLS-over-TCP transport. The minimum TLS version defaults to 1.2 and can be bumped to 1.3 with `-tls_min_version=1.3`.

### Health Probe

The server provides a plain HTTP health endpoint on a dedicated port for Kubernetes probes and cloud load balancer health checks. This endpoint is independent of the LSP transport: it works the same in both `-mode=tcp` and `-mode=websocket`, with or without `-tls`, and never requires authentication.

```
GET /healthz   →  200 "ok"           when the LSP listener is accepting connections
GET /healthz   →  503 "not ready..."   while the listener has not yet started
```

By default, the health probe server is disabled. To enable it, specify the address to listen on using `-health_addr`. For example, in a containerized environment, bind to all interfaces so that kubelet and cloud load balancers can reach the probe:

```bash
s2-language-server -health_addr=:8081
```

In the default Docker image, the readiness probe listens on `127.0.0.1:8081`. Adjust the address as needed for your deployment. To disable the probe server entirely, pass `-health_addr=""`.

The probe handler does not log individual requests—this avoids flooding the LSP log stream with frequent kubelet polls. Probe traffic is always plain HTTP, so certificate rotation on the LSP transport never affects the health endpoint. For a reference Kubernetes deployment (compatible with EKS, GKE, and AKS) that configures `livenessProbe`, `readinessProbe`, and cloud LB health checks at this endpoint, see [`deploy/k8s.yaml`](deploy/k8s.yaml).

### Resource Tuning

The `-max_files_with_states` and `-max_states_per_file` flags control memory usage. Lower values reduce memory consumption at the cost of re-parsing files more often. The defaults (`100` files, `1000` states per file) are suitable for most workloads.

## Grammar Files

The grammar directory must contain a default DDL and DML grammar, and may optionally contain additional grammars targeted at specific SingleStore engine versions:

- `ddl.bin` — compiled default DDL grammar (required)
- `dml.bin` — compiled default DML grammar (required)
- `ddl_<version>.bin` / `dml_<version>.bin` — optional version-specific grammars (e.g. `ddl_9.0.30.bin`, `dml_9.0.30.bin`)

The default grammars are mandatory; the server fails to start if either is missing. Version-specific grammars are optional and use the form `<grammar>_<semver>.bin`, where `<semver>` matches the value returned by SingleStore's `@@memsql_version`.

### Version selection at runtime

When the client provides database connection details in `initializationOptions.database`, the server queries `@@memsql_version` from the connected engine and picks the closest matching grammar for both DDL and DML, independently:

1. If a grammar file matches the engine version exactly, it is used.
2. Otherwise the server looks for a grammar with the same major and minor version, preferring the highest patch version that is less than or equal to the engine's patch. If none qualify, the lowest patch version greater than the engine's patch is used.
3. If no grammar shares the engine's major and minor version, the default grammar (`ddl.bin` / `dml.bin`) is used.

If no database connection is configured (or the connection fails), the server falls back to the default grammar.

## Official Clients

| Client | Platform | Marketplace | Repository |
|---|---|---|---|
| SingleStore VS Code | VS Code | [Extension](https://marketplace.visualstudio.com/items?itemName=singlestore.singlestore-vscode&ssr=false#overview) | [GitHub](https://github.com/singlestore-labs/singlestore-vscode) |
| SingleStore Vim | Vim 8+ / Neovim | — | [GitHub](https://github.com/singlestore-labs/singlestore-vim) |
| SingleStore JetBrains | JetBrains IDEs [^jb] | [Plugin](https://plugins.jetbrains.com/plugin/32345-singlestore-language-server-client/versions/stable/) | [GitHub](https://github.com/singlestore-labs/singlestore-intellij-plugin) |
| SingleStore Emacs | Emacs | — | [GitHub](https://github.com/singlestore-labs/singlestore-emacs) |

[^jb]: Supported paid IDEs: IntelliJ IDEA Ultimate, WebStorm, PyCharm Professional, GoLand, CLion, Rider, DataGrip, RubyMine.


## Client Requirements

Language server clients **must** supply two groups of parameters inside `initializationOptions` when sending the [`initialize`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#initialize) request.

### Database Connection Credentials

The server connects to a SingleStore engine on startup to introspect schemas, tables, columns, and other metadata used for completions.  
Provide connection details under `initializationOptions.database`:

| Field      | JSON key     | Type    | Required | Description |
|------------|--------------|---------|----------|-------------|
| Host       | `host`       | string  | yes      | Hostname or IP address of the SingleStore engine (e.g. `"svc-xxx.svc.singlestore.com"`) |
| Port       | `port`       | integer | yes      | MySQL-protocol port (typically `3306`) |
| Username   | `username`   | string  | yes      | Database user with at least read access to the schemas that should be completed |
| Password   | `password`   | string  | yes      | Password for the database user |
| Database   | `database`   | string  | no       | Default database |
| SSL        | `ssl`        | boolean | no       | When `true`, the connection will use TLS. Defaults to `false` |

> **Note:** If `initializationOptions.database` is not provided or the connection to the database cannot be established, the language server will still start and operate normally. However, completions based on the actual database schema (tables, columns, procedures, etc.) will not be available. The server will notify the client about the connection issue via a `window/showMessage` warning.

### Client Information

Provide the name and version of the language client under `initializationOptions.client`:

| Field   | JSON key  | Type   | Required | Description |
|---------|-----------|--------|----------|-------------|
| Name    | `name`    | string | yes      | Human-readable client name (e.g. `"vscode-singlestore"`) |
| Version | `version` | string | yes      | SemVer version string of the client (e.g. `"1.2.0"`) |

### Full Example

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "processId": 12345,
    "rootUri": null,
    "capabilities": {},
    "initializationOptions": {
      "database": {
        "host": "127.0.0.1",
        "port": 3306,
        "username": "root",
        "password": "s3cret",
        "database": "mydb",
        "ssl": false
      },
      "client": {
        "name": "vscode-singlestore",
        "version": "1.2.0"
      }
    }
  }
}
```

### Updating Database Connection at Runtime

After initialization, clients can change the database the server is connected to (for example, when a user switches workspaces or rotates credentials) without restarting the language server. Send a [`workspace/didChangeConfiguration`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#workspace_didChangeConfiguration) notification with the updated settings.

The `settings` payload accepts the same `database` block used during initialization, plus an optional `client` block. When `client` is omitted, the server reuses the language client identity supplied in the original `initialize` call. If `database.host` is not set, the language server will only provide grammar-based suggestions without retrieving the actual database schema.

If the notification is sent with an empty `settings` field (missing, `null`, or `{}`), the server ignores it entirely and performs no reconfiguration. This allows clients that broadcast generic `workspace/didChangeConfiguration` events to do so without disrupting the active database connection.

| Settings field | Required | Description |
|----------------|----------|-------------|
| `database`     | yes      | Same shape as `initializationOptions.database` |
| `client`       | no       | Override the language client name/version |

On success the server stops the previous schema cache updater, opens a fresh connection, restarts the cache updater, and clears any prior connection error. On failure the error is logged on the server side and the previously running cache updater is left stopped. Because `workspace/didChangeConfiguration` is a notification, no response is returned to the client.

#### Example

```json
{
  "jsonrpc": "2.0",
  "method": "workspace/didChangeConfiguration",
  "params": {
    "settings": {
      "database": {
        "host": "svc-new.svc.singlestore.com",
        "port": 3306,
        "username": "analyst",
        "password": "n3w-s3cret",
        "database": "analytics",
        "ssl": true
      }
    }
  }
}
```

### Using the language server for query validation

The language server provides a dedicated method for validating SQL queries without executing them against a live database. This is exposed via the custom LSP request `queryChecker/validateQuery`. Clients can use this method to check the syntax and structure of a query, receive error locations, and get suggestions for valid tokens at the point of failure.

To validate a query, send a JSON-RPC request with the following structure:

```JSON
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "queryChecker/validateQuery",
  "params": {
    "databaseVersion": "9.0.30",   // optional: engine version for grammar selection
    "query": "LOAD DATA CONCURRENT LOCAL FILE 'myfile'" // the SQL query to validate
  }
}
```

The response will indicate whether the query is valid, and if not, will provide the error token, its position, and a list of alternative tokens that would be valid at that point:

```JSON
{
  "isValid": false, // indicates wheter the query is grammatically correct
  "errorTokenPosition": { "line": 0, "character": 27 }, // position of the first incorrect token
  "errorToken": "FILE", // first incorrect token
  "alternativeTokens": { // list of alternatives, expected instead of the incorrect tocken
    "isIncomplete": false,
    "items": [
      {
        "label": "INFILE",
        ...
      },
      ...
    ]
  }
}
```

If the query is valid, `isValid` will be true and the other fields will be omitted.
If invalid, `errorToken` and `errorTokenPosition` point to the problematic token, and `alternativeTokens` suggests valid completions.
This method is useful for editor integrations, CI pipelines, and any tool that needs fast, offline query validation using the same grammar as the language server’s completion engine.
