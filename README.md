# SingleStore SQL Language Server

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
| `-generate_bin` | `false` | Generate binary grammar files from source grammar and exit |
| `-tls` | `false` | Enable TLS on the listener (`wss://` for `-mode=websocket`, TLS-over-TCP for `-mode=tcp`) |
| `-tls_cert` | `""` | Path to PEM-encoded TLS server certificate (required when `-tls`) |
| `-tls_key` | `""` | Path to PEM-encoded TLS server private key (required when `-tls`) |
| `-tls_min_version` | `"1.2"` | Minimum TLS version: `1.2` or `1.3` |

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
|--------|----------|-------------|------------|
| SingleStore VS Code | VS Code | [Extension](https://marketplace.visualstudio.com/items?itemName=singlestore.singlestore-vscode&ssr=false#overview) | [GitHub](https://github.com/singlestore-labs/singlestore-vscode) |
| SingleStore Vim | Vim 8+ / Neovim | | [GitHub](https://github.com/singlestore-labs/singlestore-vim) |

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

The `settings` payload accepts the same `database` block used during initialization, plus an optional `client` block. When `client` is omitted, the server reuses the language client identity supplied in the original `initialize` call. If `database.host` is empty, the notification is ignored.

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
