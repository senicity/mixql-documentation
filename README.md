```

███╗   ███╗██╗██╗  ██╗ ██████╗ ██╗     
████╗ ████║██║╚██╗██╔╝██╔═══██╗██║     
██╔████╔██║██║ ╚███╔╝ ██║   ██║██║     
██║╚██╔╝██║██║ ██╔██╗ ██║▄▄ ██║██║     
██║ ╚═╝ ██║██║██╔╝ ██╗╚██████╔╝███████╗
╚═╝     ╚═╝╚═╝╚═╝  ╚═╝ ╚══▀▀═╝ ╚══════╝
                                       
// -- Powered by:

┏┓┏┓┳┓┳┏┓┳┏┳┓┓┏
┗┓┣ ┃┃┃┃ ┃ ┃ ┗┫
┗┛┗┛┛┗┻┗┛┻ ┻ ┗┛
               
// --> https://senicity.com
// --
```

## MixQL Server

A TCP-based query language server for hashing, salting, and one-way encryption operations. Built with C++17, OpenSSL, and LevelDB.

---

## Prerequisites

- CMake 3.10+
- C++17 compiler (g++ or clang++)
- OpenSSL development libraries
- LevelDB development libraries
- Compression libraries: zlib, bzip2, snappy, lz4, zstd
- gflags

### Install on Ubuntu/Debian

```bash
sudo apt-get install build-essential cmake libssl-dev libleveldb-dev \
  libgflags-dev libsnappy-dev libbz2-dev liblz4-dev libzstd-dev
```

### Install on Alpine

```bash
apk add build-base cmake openssl3-dev leveldb-dev gflags-dev \
  snappy-dev bzip2-dev lz4-dev zstd-dev
```

### Install on macOS

```bash
brew install cmake openssl leveldb gflags snappy lz4 zstd
```

---

## Build

```bash
mkdir build && cd build
cmake ..
make
```

The binary `mixql` is produced in the build directory.

---

## Configuration

Create a `config.ini` file in the working directory (see `config/config.example.ini`):

```ini
[server]
port=7272
dbname=mixql
```

Alternatively, use environment variables (these override file values):

```bash
export SERVER_PORT=7272
export SERVER_DBNAME=mixql
```

| Parameter | Env Var | Default | Description |
|---|---|---|---|
| `port` | `SERVER_PORT` | `7272` | TCP listen port |
| `dbname` | `SERVER_DBNAME` | `datastore` | LevelDB database name |

---

## Running

```bash
./mixql
```

The server prints the NOTICE file and begins listening on the configured port.

Send `SIGINT` (Ctrl+C) or `SIGTERM` to gracefully shut down the server.

---

## Docker

### Alpine (smaller image)

```bash
docker build -f Dockerfile.alpine -t mixql:alpine .
docker run -p 7272:7272 mixql:alpine
```

### Ubuntu

```bash
docker build -f Dockerfile.ubuntu -t mixql:ubuntu .
docker run -p 7272:7272 mixql:ubuntu
```

---

## Usage

Connect via any TCP client. The protocol is plain text: send a query followed by a newline, then any parameters (one per line).

### Connecting with netcat

```bash
echo -e "QUERY\nPARAM1\nPARAM2" | nc localhost 7272
```

### Connecting with a TCP socket (Python example)

```python
import socket

def mixql(query, params=None):
    payload = query + "\n" + ("\n".join(params) if params else "")
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.connect(("localhost", 7272))
        s.sendall(payload.encode())
        return s.recv(4096).decode()
```

---

## Query Language Reference

### SELECT — Hash Expressions

```sql
SELECT <expression> AS hash;
SELECT <expression> AS hash UPPERCASE;
```

Evaluates nested functions and returns the result Base64-encoded.

**Available functions**:

| Function | Description |
|---|---|
| `SHA1(x)` | SHA-1 hash (hex) |
| `MD5(x)` | MD5 hash (hex) |
| `BASE64_ENCODE(x)` | Base64 encode |
| `CONCAT(a, b, ...)` | Concatenate values |
| `NOW()` | Current Unix timestamp |

**Examples**:

```bash
# Simple SHA1 hash
echo -e "SELECT SHA1(:input) AS hash\nhello" | nc localhost 7272

# Nested: SHA1 of a concatenation with a timestamp
echo -e "SELECT SHA1(CONCAT(:user, NOW())) AS hash\nadmin" | nc localhost 7272

# MD5 with uppercase output
echo -e "SELECT MD5(:data) AS hash UPPERCASE\nmy_secret" | nc localhost 7272

# Multiple parameters
echo -e "SELECT SHA1(CONCAT(:a, :b)) AS hash\nfoo\nbar" | nc localhost 7272
```

### CREATE UUID

```bash
echo -e "CREATE UUID\n" | nc localhost 7272
# Output: a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d
```

### CREATE SALT

```bash
# Single salt (16 chars)
echo -e "CREATE SALT\n" | nc localhost 7272

# 5 salts of 32 characters each
echo -e "CREATE SALT LIMIT 5 LENGTH 32\n" | nc localhost 7272

# SHA-1 hashed salt (fixed 40-char output)
echo -e "CREATE SALT SHA\n" | nc localhost 7272

# Combined options
echo -e "CREATE SALT LIMIT 10 LENGTH 24 SHA\n" | nc localhost 7272
```

### CREATE KEY

```bash
# Single key
echo -e "CREATE KEY\n" | nc localhost 7272

# 5 keys
echo -e "CREATE KEY LIMIT 5\n" | nc localhost 7272
```

Keys are generated as: `random salt → SHA-1 → hex → Base64`.

### STORE — Persistent Named Queries

```bash
# Store a query for reuse
echo -e "SELECT SHA1(:password) AS hash STORE AS hash_password\n" | nc localhost 7272

# List all stored queries
echo -e "STORE LIST\n" | nc localhost 7272

# View a stored query
echo -e "STORE SELECT hash_password\n" | nc localhost 7272

# Execute a stored query with parameters
echo -e "STORE USE hash_password\nmy_secret_pass" | nc localhost 7272

# Delete a stored query
echo -e "STORE DELETE hash_password\n" | nc localhost 7272
```

---

## Error Responses

| Response | Meaning |
|---|---|
| `INVALID_INPUT` | Missing newline separator or unrecognized command |
| `INVALID_QUERY` | SELECT query syntax did not match expected pattern |
| `INVALID_FUNCTION` | Unknown function name in expression |
| `Query not found: '<name>'` | STORE SELECT/USE referenced a non-existent query |

---

## Architecture

See [ARCHITECTURE.md](ARCHITECTURE.md) for system design details and [AGENTS.md](AGENTS.md) for component documentation.

---

## License

Copyright ©2026, Senicity Ltd. See `NOTICE` for details.
