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

# MixQL Documentation

MixQL is a TCP-based query language server for hashing, salting, encryption,
storage, and key generation. The server is built with C++17, OpenSSL, and
LevelDB.

The MixQL server speaks a plain-text protocol over TCP. TLS is normally
terminated by an Nginx `stream` proxy in production. The server itself
continues to listen on plain TCP behind the proxy.

## Docker Quick Start

### Pull an image

```bash
docker pull senicity/mixql:latest
# or

docker pull senicity/mixql:alpine
```

The Alpine image is the recommended small production image. The Ubuntu image
is available when broader system-library compatibility is needed.

### Run with temporary local storage

```bash
docker run --rm \
  --name mixql \
  -p 127.0.0.1:7272:7272 \
  senicity/mixql:latest
```

This binds MixQL only to the server's loopback interface. It is suitable when
Nginx runs on the same host and forwards traffic to port `7272`.

### Run with authentication and persistent storage

Create a configuration file on the host:

```bash
sudo mkdir -p /opt/mixql /data/mixql/db

sudo tee /opt/mixql/config.ini >/dev/null <<'EOF'
port=7272
dbname=/data/datastore
auth_enabled=true
auth_users=admin:REPLACE_WITH_A_STRONG_PASSWORD
enc_key=REPLACE_WITH_A_LONG_RANDOM_ENCRYPTION_KEY
EOF
```

Start the container:

```bash
docker run -d \
  --name mixql \
  --restart unless-stopped \
  -p 127.0.0.1:7272:7272 \
  -v /opt/mixql/config.ini:/app/config.ini:ro \
  -v /data/mixql/db:/data \
  senicity/mixql:latest
```

The LevelDB database will be stored on the host at:

```text
/data/mixql/db/datastore.db
```

The server appends `.db` to the configured `dbname`. The container reads the
runtime configuration from `/app/config.ini`.

Check the container:

```bash
docker logs -f mixql
```

Test it locally on the Docker host:

```bash
printf 'AUTH admin:REPLACE_WITH_A_STRONG_PASSWORD\nCREATE UUID\n' |
  nc 127.0.0.1 7272
```

### Docker configuration overrides

Environment variables override values loaded from `config.ini`:

```bash
docker run -d \
  --name mixql \
  -p 127.0.0.1:7272:7272 \
  -v /data/mixql/db:/data \
  -e SERVER_PORT=7272 \
  -e SERVER_DBNAME=/data/datastore \
  -e SERVER_AUTH_ENABLED=true \
  -e SERVER_ENC_KEY='REPLACE_WITH_A_LONG_RANDOM_ENCRYPTION_KEY' \
  senicity/mixql:latest
```

Supported environment variables are:

| Variable | Description |
|---|---|
| `SERVER_PORT` | TCP listen port |
| `SERVER_DBNAME` | Database path prefix; `.db` is appended |
| `SERVER_AUTH_ENABLED` | `true` or `false` |
| `SERVER_ENC_KEY` | Default encryption key |

Authentication users are configured with the `auth_users` setting in the
configuration file. Credentials are stored as plain text by the current
server implementation. Use a dedicated configuration file with restricted
permissions and do not commit it to source control.

## Nginx TLS Deployment

MixQL is a raw TCP service, so use Nginx's `stream` module rather than an HTTP
reverse proxy.

The main `nginx.conf` can include stream configurations like this:

```nginx
stream {
    include /etc/nginx/stream.d/*.conf;
}
```

Create `/etc/nginx/stream.d/mixql.conf`:

```nginx
upstream mixql_backend {
    server 127.0.0.1:7272;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name mixql.example.com;

    ssl_certificate /etc/letsencrypt/live/mixql.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mixql.example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    ssl_session_cache shared:MixQL_SSL:10m;
    ssl_session_timeout 10m;

    proxy_pass mixql_backend;
    proxy_connect_timeout 5s;
    proxy_timeout 300s;
}
```

Validate and reload Nginx:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Test the TLS endpoint:

```bash
openssl s_client \
  -connect mixql.example.com:443 \
  -servername mixql.example.com \
  </dev/null
```

If port `443` is already used by an HTTP server on the same address, use a
dedicated TLS port or a dedicated IP. A normal HTTP reverse proxy cannot
forward the MixQL protocol.

## Protocol

Each request contains a query line followed by one parameter per line:

```text
SELECT SHA256(:input) AS hash
hello
```

When authentication is enabled, the first line must be an authentication
command:

```text
AUTH username:password
SELECT SHA256(:input) AS hash
hello
```

The server handles one request per TCP connection and closes the connection
after returning the response.

### Netcat examples

Unauthenticated local request:

```bash
printf 'CREATE UUID\n' | nc 127.0.0.1 7272
```

Authenticated local request:

```bash
printf 'AUTH admin:YOUR_PASSWORD\nCREATE UUID\n' |
  nc 127.0.0.1 7272
```

The TLS endpoint must be tested with a TLS-capable client, not plain `nc`:

```bash
printf 'AUTH admin:YOUR_PASSWORD\nCREATE UUID\n' |
  openssl s_client -quiet \
    -connect mixql.example.com:443 \
    -servername mixql.example.com
```

## Query Language Reference

### SELECT syntax

```sql
SELECT <expression> AS hash
SELECT <expression> AS hash UPPERCASE
SELECT <expression> KEY <key> AS hash
SELECT <expression> KEY <key> SALT <s1>,<s2> PEPPER <p1>,<p2> AS hash
```

Parameters use colon-prefixed names and are supplied on following lines.
Nested expressions are evaluated from the inside out.

### Functions

| Function | Description |
|---|---|
| `SHA256(x)` | SHA-256 hash |
| `SHA512(x)` | SHA-512 hash |
| `SHA1(x)` | SHA-1 hash; legacy |
| `MD5(x)` | MD5 hash; legacy |
| `BASE64_ENCODE(x)` | Base64 encoding |
| `CONCAT(a, b, ...)` | Concatenates values |
| `NOW()` | Current Unix timestamp |
| `ENC(x)` | AES-256-CBC encryption |
| `DEC(x)` | AES-256-CBC decryption |
| `ENC_GCM(x)` | AES-256-GCM authenticated encryption |
| `DEC_GCM(x)` | AES-256-GCM authenticated decryption |
| `HMAC(key, msg)` | HMAC-SHA256 |
| `ARGON2(x)` | Argon2id password hash |
| `ARGON2_VERIFY(hash, x)` | Verifies an Argon2id password hash |

For encryption and decryption, the optional clauses are:

| Clause | Description |
|---|---|
| `KEY <key>` | Overrides the configured encryption key |
| `SALT <s1>,<s2>,...` | Applies layered encryption passes |
| `PEPPER <p1>,<p2>,...` | Interleaves pepper values between characters |

Examples:

```bash
# SHA-256
printf 'SELECT SHA256(:input) AS hash\nhello\n' | nc localhost 7272

# SHA-512
printf 'SELECT SHA512(:input) AS hash\nhello\n' | nc localhost 7272

# Nested expression
printf 'SELECT SHA1(CONCAT(:user, NOW())) AS hash\nadmin\n' |
  nc localhost 7272

# Multiple parameters
printf 'SELECT SHA1(CONCAT(:a, :b)) AS hash\nfoo\nbar\n' |
  nc localhost 7272

# HMAC
printf 'SELECT HMAC(:key, :msg) AS hash\nsecret\nmessage\n' |
  nc localhost 7272

# Argon2id
printf 'SELECT ARGON2(:password) AS hash\nmy-password\n' |
  nc localhost 7272

# GCM encryption
printf 'SELECT ENC_GCM(:input) AS hash\nhello\n' | nc localhost 7272

# Encryption with a custom key
printf 'SELECT ENC(:input) KEY mysecretkey123 AS hash\nhello\n' |
  nc localhost 7272

# Layered encryption with salt and pepper
printf 'SELECT ENC(:input) KEY :key SALT :s1,:s2 PEPPER :p1,:p2 AS hash\nmessage\nmykey\nsalt1\nsalt2\npep1\npep2\n' |
  nc localhost 7272
```

### CREATE commands

```bash
# UUID
printf 'CREATE UUID\n' | nc localhost 7272

# Salts
printf 'CREATE SALT\n' | nc localhost 7272
printf 'CREATE SALT LIMIT 5 LENGTH 32\n' | nc localhost 7272
printf 'CREATE SALT SHA\n' | nc localhost 7272

# Keys
printf 'CREATE KEY\n' | nc localhost 7272
printf 'CREATE KEY LIMIT 5\n' | nc localhost 7272
```

Keys are generated from random data, SHA-1 hashing, hexadecimal encoding, and
Base64 encoding.

### STORE commands

Stored queries use LevelDB and persist when the database directory is mounted:

```bash
# Store a query
printf 'SELECT SHA1(:password) AS hash STORE AS hash_password\n' |
  nc localhost 7272

# List stored queries
printf 'STORE LIST\n' | nc localhost 7272

# Retrieve a stored query
printf 'STORE SELECT hash_password\n' | nc localhost 7272

# Execute a stored query
printf 'STORE USE hash_password\nmy_secret_pass\n' | nc localhost 7272

# Delete a stored query
printf 'STORE DELETE hash_password\n' | nc localhost 7272
```

## Command-Line Client

The Bash CLI is in the `mixql-cli` package. Start it with plain TCP:

```bash
bash mixql.sh
```

Use the TLS endpoint through Nginx:

```bash
bash mixql.sh \
  --host mixql.example.com \
  --port 443 \
  --tls
```

The CLI uses `ncat --ssl` when Ncat is installed. If Ncat is unavailable, it
uses the OpenSSL TLS transport built into `mixql.sh`. Plain TCP remains the
default. The CLI verifies the TLS certificate and hostname by default.

Available connection options:

```text
-h, --host HOST
-p, --port PORT
-t, --tls
    --no-tls
```

When authentication is enabled, the CLI detects the `AUTH_FAILED` response and
prompts for credentials.

## PHP SDK

Install the package with Composer:

```bash
composer require senicity/mixql
```

Plain TCP:

```php
$mixql = new MixQL(
    ['timeout' => 30],
    'localhost',
    7272
);

$result = $mixql
    ->sha256()
    ->bind(['hello'])
    ->execute();

echo $result;
```

TLS through Nginx:

```php
$mixql = new MixQL(
    [
        'timeout' => 30,
        'connect_timeout' => 10,
        'tls' => true,
    ],
    'mixql.example.com',
    443
);

$result = $mixql
    ->createUUID()
    ->auth('admin', 'YOUR_PASSWORD')
    ->execute();

echo $result;
```

PHP uses its built-in `tls://` stream transport. Certificate and hostname
verification are enabled by default. Optional TLS settings include
`tls_server_name`, `tls_verify`, `tls_ca_file`, and `tls_ca_path`.

## Node.js SDK

Install the package:

```bash
npm install @senicity/mixql
```

Plain TCP:

```js
const MixQL = require('@senicity/mixql');

const mixql = new MixQL(
  { timeout: 30 },
  'localhost',
  7272
);

const result = await mixql
  .sha256()
  .bind(['hello'])
  .execute();

console.log(result.toString());
```

TLS through Nginx:

```js
const mixql = new MixQL(
  {
    timeout: 30,
    connect_timeout: 10,
    tls: true,
  },
  'mixql.example.com',
  443
);

const result = await mixql
  .createUUID()
  .auth('admin', 'YOUR_PASSWORD')
  .execute();

console.log(result.toString());
```

The Node.js SDK uses the built-in `tls` module. Certificate and hostname
verification are enabled by default. Optional settings include
`tls_server_name`, `tls_verify`, `tls_ca_file`, and `tls_ca`.

## Local SDK Tests

### PHP test runner

From the `mixql-php` directory, plain TCP uses `localhost:7272` by default:

```bash
php bin/run.php Query CreateUUID
php bin/run.php Query SHA256 --bind=hello
php bin/run.php Query CreateUUID --auth=admin:YOUR_PASSWORD
```

Test the public TLS endpoint:

```bash
php bin/run.php Query CreateUUID \
  --host=mixql.example.com \
  --port=443 \
  --tls \
  --auth=admin:YOUR_PASSWORD
```

Supported connection flags include `--host`, `--port`, `--tls`,
`--tls-verify=false`, `--tls-server-name`, `--ca-file`, `--ca-path`,
`--timeout`, and `--connect-timeout`.

### Node.js SDK

The Node.js SDK does not require a separate test runner. Run a small script:

```bash
node <<'NODE'
const MixQL = require('@senicity/mixql');

(async () => {
  const result = await new MixQL(
    { tls: true, timeout: 5 },
    'mixql.example.com',
    443
  )
    .createUUID()
    .auth('admin', 'YOUR_PASSWORD')
    .execute();

  console.log(result.toString());
})().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
NODE
```

## Error Responses

| Response | Meaning |
|---|---|
| `AUTH_FAILED` | Authentication is required or credentials are invalid |
| `INVALID_INPUT` | Missing newline separator or unrecognized command |
| `INVALID_QUERY` | SELECT syntax did not match the expected pattern |
| `INVALID_FUNCTION` | Unknown function name |
| `Query not found: '<name>'` | A STORE command referenced a missing query |

## Security Notes

- Keep the MixQL container bound to `127.0.0.1` when Nginx runs on the same host.
- Use Nginx `stream` TLS termination for public TLS access.
- Do not use an HTTP reverse proxy for the raw MixQL protocol.
- Keep certificate and hostname verification enabled in clients.
- Change all example passwords and encryption keys.
- Authentication credentials are currently stored in plain text in the server configuration.
- Do not expose the plain TCP port publicly when TLS termination is handled by Nginx.
- Use `ENC_GCM` for authenticated encryption where possible.
- Treat SHA-1 and MD5 as legacy functions.

## Links

- [Docker Hub](https://hub.docker.com/r/senicity/mixql)
- [MixQL CLI](https://github.com/senicity/mixql-cli)
- [MixQL PHP SDK](https://github.com/senicity/mixql-php)
- [MixQL Node.js SDK](https://github.com/senicity/mixql-js)
- [Senicity](https://senicity.com)

## License

Copyright ©2026, Senicity Ltd. See `NOTICE` for details.
