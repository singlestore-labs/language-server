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

Extracts `s2-language-server` binary and `grammar_dir/` (containing `ddl.bin` and `dml.bin`) into the current directory. Place them wherever you prefer and reference `grammar_dir` with the `-grammar_dir` flag.

### Windows (zip)

Extract `s2-language-server_<version>_windows_amd64.zip`. Contains `s2-language-server.exe` and `grammar_dir\` with the grammar binary files.

## Usage

```bash
s2-language-server [flags]
```

### Flags

| Flag | Default | Description |
|------|---------|-------------|
| `-mode` | `tcp` | Communication protocol: `tcp` or `websocket` |
| `-addr` | `:8080` | Address to listen on |
| `-grammar_dir` | `""` | Path to the directory containing grammar binary files (`ddl.bin`, `dml.bin`) |
| `-max_files_with_states` | `100` | Maximum number of files to keep parser states for |
| `-max_states_per_file` | `1000` | Maximum number of parser states to keep per file |
| `-allow_all_origins` | `true` | Allow WebSocket connections from any origin (WebSocket mode only) |
| `-generate_bin` | `false` | Generate binary grammar files from source grammar and exit |

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

### Resource Tuning

The `-max_files_with_states` and `-max_states_per_file` flags control memory usage. Lower values reduce memory consumption at the cost of re-parsing files more often. The defaults (`100` files, `1000` states per file) are suitable for most workloads.

## Grammar Files

The grammar directory must contain two binary files:

- `ddl.bin` — compiled DDL grammar
- `dml.bin` — compiled DML grammar

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


