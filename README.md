# MoxSignal

[中文文档](./README.zh-CN.md)

MoxSignal is the WebRTC signaling service for MoxChat voice calls, video calls, live sessions, and group SFU negotiation. HTTP APIs handle call setup and authenticated control commands; WebSocket APIs carry real-time offer, answer, ICE, hangup, and SFU events.

Service source code and target specifications are maintained in the main Mox source repository under `spec/signaling/moxsignal/`. This repository contains deployment documentation and distributable artifacts only.

## MoxChat Clients

- iOS: [Download on the App Store](https://apps.apple.com/us/app/moxchat/id6775016915)
- Web: [Open MoxChat](https://app.ponzs.com)

## Get the Files from GitHub

This repository provides deployment documentation, prebuilt binaries, and Lazycat LPK packages. You do not need Go or a local build. The current package version is `1.3.0`; see [build information](./BUILD-INFO.md) for the source revision and build timestamp, and [SHA256SUMS](./SHA256SUMS) for file digests.

### Clone the Release Repository

With Git installed, run these commands on Linux, macOS, or Windows PowerShell:

```sh
git clone --depth 1 https://github.com/MoxChat/MoxSignal.git
cd MoxSignal
```

Verify the files after cloning and before editing configuration. On Linux:

```sh
sha256sum --check SHA256SUMS
```

On macOS:

```sh
shasum -a 256 --check SHA256SUMS
```

### Download Only the Files for Your Platform

Linux x64 example, using a new deployment directory:

```sh
mkdir moxsignal-release
cd moxsignal-release
curl -fL --retry 3 -o moxsignal-linux-amd64 https://raw.githubusercontent.com/MoxChat/MoxSignal/main/moxsignal-linux-amd64
curl -fL --retry 3 -o SHA256SUMS https://raw.githubusercontent.com/MoxChat/MoxSignal/main/SHA256SUMS
awk '$2 == "moxsignal-linux-amd64" {print}' SHA256SUMS | sha256sum --check
chmod +x ./moxsignal-linux-amd64
```

For Linux arm64, replace `linux-amd64` with `linux-arm64`. On macOS, use the matching `darwin-*` filename from the table below and replace the checksum command with `shasum -a 256 --check`.

For Windows x64, download and verify the files in a new directory using PowerShell:

```powershell
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/MoxChat/MoxSignal/main/moxsignal-windows-amd64.exe" -OutFile ".\moxsignal-windows-amd64.exe"
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/MoxChat/MoxSignal/main/SHA256SUMS" -OutFile ".\SHA256SUMS"
$expected = ((Get-Content .\SHA256SUMS | Select-String '  moxsignal-windows-amd64\.exe$').Line -split '\s+')[0]
if ((Get-FileHash .\moxsignal-windows-amd64.exe -Algorithm SHA256).Hash -ne $expected) { throw "SHA-256 verification failed" }
```

For Windows arm64, replace `windows-amd64` with `windows-arm64`. Individual binary downloads do not include `.env`; set environment variables as shown in the deployment examples below. Start or install the service only after verification succeeds.

These direct URLs read the `main` branch. To pin a version, replace `main` in every download URL with the same release repository commit SHA. The source revision in the build information belongs to the main source repository and cannot be used in these URLs. If the branch changes during download and verification fails, pin one commit and download the files again.

## Release Files

Use the GitHub commands above to obtain the files that match your target platform:

| Target | File |
| --- | --- |
| Lazycat MicroServer | `moxsignal.lpk` |
| Linux x64 | `moxsignal-linux-amd64` |
| Linux arm64 | `moxsignal-linux-arm64` |
| macOS Intel | `moxsignal-darwin-amd64` |
| macOS Apple Silicon | `moxsignal-darwin-arm64` |
| Windows x64 | `moxsignal-windows-amd64.exe` |
| Windows arm64 | `moxsignal-windows-arm64.exe` |

## Lazycat MicroServer Deployment

1. Use `moxsignal.lpk` from your clone, or download and verify it in a new directory (Linux example; on macOS, replace `sha256sum` with `shasum -a 256`):

```sh
curl -fL --retry 3 -o moxsignal.lpk https://raw.githubusercontent.com/MoxChat/MoxSignal/main/moxsignal.lpk
curl -fL --retry 3 -o SHA256SUMS https://raw.githubusercontent.com/MoxChat/MoxSignal/main/SHA256SUMS
awk '$2 == "moxsignal.lpk" {print}' SHA256SUMS | sha256sum --check
```

2. Install it from the Lazycat app UI, or with the CLI:

```sh
lzc-cli app install moxsignal.lpk
```

3. Open the app at the assigned `moxsignal` subdomain.
4. Check `https://<moxsignal-host>/healthz`.

The LPK includes the MoxSignal app process and a PostgreSQL service. Group SFU media uses UDP port `8982`, the same numeric port used by the HTTP/WebSocket listener. The protocols are separate, so production deployments must expose both `8982/TCP` and `8982/UDP`.

A normal HTTP reverse proxy or CDN only forwards TCP. When using Cloudflare, the hostname must be DNS-only with a directly reachable UDP path, or use a Cloudflare product that explicitly supports UDP proxying; the ordinary orange-cloud HTTP proxy is not sufficient for SFU media.

## Linux Deployment

These commands assume you are in the binary directory and PostgreSQL is installed and running. Run the SQL in a PostgreSQL session with permission to create users and databases. Replace `change-me` in both the SQL and the connection string with the same database password.

Create a PostgreSQL database:

```sql
CREATE USER moxsignal WITH PASSWORD 'change-me';
CREATE DATABASE moxsignal OWNER moxsignal;
```

Run the binary:

```sh
chmod +x ./moxsignal-linux-amd64
export MOXSIGNAL_ADDR=:8982
export MOXSIGNAL_DB_DSN='postgres://moxsignal:change-me@127.0.0.1:5432/moxsignal?sslmode=disable'
export MOXSIGNAL_PUBLIC_BASE_URL='https://moxsignal.example.com'
export MOXSIGNAL_SFU_UDP_PORT=8982
./moxsignal-linux-amd64
```

Use `moxsignal-linux-arm64` on arm64 hosts. If group SFU is enabled, expose the configured UDP port to clients.

## macOS Deployment

Use the matching macOS binary:

```sh
chmod +x ./moxsignal-darwin-arm64
export MOXSIGNAL_ADDR=:8982
export MOXSIGNAL_DB_DSN='postgres://moxsignal:change-me@127.0.0.1:5432/moxsignal?sslmode=disable'
export MOXSIGNAL_PUBLIC_BASE_URL='https://moxsignal.example.com'
./moxsignal-darwin-arm64
```

If macOS blocks a downloaded binary, remove the quarantine attribute:

```sh
xattr -d com.apple.quarantine ./moxsignal-darwin-arm64
```

## Windows Deployment

Create the PostgreSQL database first, then start MoxSignal from PowerShell:

```powershell
$env:MOXSIGNAL_ADDR = ":8982"
$env:MOXSIGNAL_DB_DSN = "postgres://moxsignal:change-me@127.0.0.1:5432/moxsignal?sslmode=disable"
$env:MOXSIGNAL_PUBLIC_BASE_URL = "https://moxsignal.example.com"
$env:MOXSIGNAL_SFU_UDP_PORT = "8982"
.\moxsignal-windows-amd64.exe
```

Use `moxsignal-windows-arm64.exe` on Windows arm64 hosts.

If group SFU is enabled, allow the configured UDP port through the Windows firewall and your network edge.

## Environment Variables

This release directory includes a default `.env` file. When the binary is started from this directory, MoxSignal loads `.env` automatically; set `MOXSIGNAL_ENV_FILE` to point at a different file. The default file is a deployment template, so replace `change-me` database credentials and set public URL, ICE, and SFU values for your network before production use.

| Variable | Required | Description |
| --- | --- | --- |
| `MOXSIGNAL_ADDR` | No | Network address for HTTP APIs, WebSocket signaling, health checks, and the operations page. Default: `:8982`. |
| `MOXSIGNAL_DB_DSN` | Yes | PostgreSQL connection string used for call sessions, participants, signaling events, API auth state, and schema state. |
| `MOXSIGNAL_ENV_FILE` | No | Alternate env file path. If omitted, the service searches for `.env` in the current directory and parent directories. |
| `MOXSIGNAL_PUBLIC_BASE_URL` | Recommended | Externally reachable HTTPS URL written into call invitations so peers connect to the same signaling relay. |
| `MOXSIGNAL_RTC_ICE_SERVERS` | No | JSON array of STUN/TURN servers distributed to clients for WebRTC candidate gathering. |
| `MOXSIGNAL_WS_TOKEN_TTL_SECONDS` | No | Lifetime of short-lived WebSocket access tokens issued by the secure API. Default: `900`. |
| `MOXSIGNAL_EVENT_TTL_SECONDS` | No | Retention time for undelivered signaling events before background cleanup removes them. Default: `120`. |
| `MOXSIGNAL_START_CALL_RATE_PER_MIN` | No | Per-identity rate limit for creating call sessions. Default: `30`. |
| `MOXSIGNAL_WS_MESSAGE_RATE_PER_SEC` | No | Per-WebSocket message rate limit to reduce relay abuse and runaway clients. Default: `60`. |
| `MOXSIGNAL_SFU_UDP_PORT` | SFU only | Fixed UDP media port for group SFU traffic. Expose this port when group media is enabled. |
| `MOXSIGNAL_SFU_PUBLIC_IPS` | SFU behind NAT | Comma-separated public IPs advertised in SFU ICE candidates when the host is behind NAT. |
| `MOXSIGNAL_SFU_PUBLIC_IPS_AS_HOST` | No | Set to `true` to advertise host IP candidates as public SFU candidates. Default: `false`. |
| `MOXSIGNAL_SFU_UDP4_ONLY` | No | Set to `true` to restrict SFU UDP sockets to IPv4. |
| `MOXSIGNAL_SFU_TOKEN_SECRET` | Recommended | Secret used to sign SFU join tokens. If omitted, a deployment-derived fallback is generated from the DSN. |
| `MOXSIGNAL_SFU_TOKEN_TTL_SECONDS` | No | Lifetime of SFU join tokens. Default: `120`. |
| `MOXSIGNAL_SFU_ICE_DISCONNECTED_SECONDS` | No | ICE disconnected timeout for SFU peer connections. Default: `8`. |
| `MOXSIGNAL_SFU_ICE_FAILED_SECONDS` | No | ICE failed timeout for SFU peer connections. Default: `25`. |
| `MOXSIGNAL_SFU_ICE_KEEPALIVE_SECONDS` | No | SFU ICE keepalive interval. Default: `2`. |
| `MOXSIGNAL_CHALLENGE_TTL_SECONDS` | No | Lifetime of challenge-response authentication codes. Default: `60`. |
| `MOXSIGNAL_AUTH_FAIL_WINDOW_MINUTES` | No | Time window used to count failed authentication attempts per source IP. Default: `30`. |
| `MOXSIGNAL_AUTH_FAIL_BAN_MINUTES` | No | Temporary ban duration after a source IP exceeds the failure threshold. Default: `30`. |
| `MOXSIGNAL_AUTH_FAIL_BAN_THRESHOLD` | No | Number of failed authentication attempts allowed in the window before banning the source IP. Default: `10`. |

Example ICE configuration:

```sh
export MOXSIGNAL_RTC_ICE_SERVERS='[{"urls":["stun:stun.example.com:3478"]},{"urls":["turn:turn.example.com:3478?transport=udp"],"username":"user","credential":"secret"}]'
```

## Health and Operations

- `GET /healthz` verifies that the HTTP process is reachable.
- `POST /api/secure/health` verifies authenticated signaling functionality, database access, `wsPath`, ICE config, and SFU media config.
- `GET /` opens the operations/status page.
- `GET /status.json` returns a status snapshot.
- `GET /status/stream` streams live status updates.

Expose the service through HTTPS in production, and make sure your reverse proxy supports WebSocket upgrade. MoxChat clients should use the externally reachable signaling URL, for example `https://moxsignal.example.com`.

## Update an Existing Deployment

Back up the database, persistent data, and runtime configuration, then stop the old process. Keep real configuration and data outside the release repository; the included `.env` is a template, so updates should not overwrite your local configuration. For a Git clone with a clean working tree:

```sh
git pull --ff-only
```

Verify `SHA256SUMS` again (Linux: `sha256sum --check SHA256SUMS`; macOS: `shasum -a 256 --check SHA256SUMS`), then start the new binary with your existing runtime configuration. For individual downloads, download the binary and checksum file from the same commit again. For LPK deployments, run the installation command again. Do not start the service if file verification fails.

After startup, check the actual listening port from another terminal (default: `8982`):

```sh
curl -f http://127.0.0.1:8982/healthz
```

Expect HTTP 200. Then check `/healthz` through the public service URL and verify the corresponding feature in MoxChat. `/healthz` only confirms that the HTTP process is reachable; it does not replace database, mesh, file transfer, SFU media, or APNs delivery validation.
