# Gomuks

## Frontend Debugging Setup

### Architecture

Three processes run simultaneously for frontend dev/debug:

| Component | Default Address | Purpose |
|---|---|---|
| Go backend (`gomuks`) | `localhost:29325` | Matrix client, API, websocket |
| Vite dev server | `localhost:5173` | Serves frontend with HMR, proxies `/_gomuks/*` to backend |
| Browser (Brave/Chrome) | CDP on `localhost:9222` | Remote debugging via Chrome DevTools Protocol |

### Prerequisites

- The flake devShell (`nix develop`) provides Go tooling but NOT Node.js
- Use `nix shell nixpkgs#nodejs` for npm commands
- Generated files are required before building: `web/src/api/types/stdcommands.json` (from Go codegen)
- The Go binary embeds `web/dist/` — for dev mode, a placeholder suffices since Vite serves the frontend

### Setup Steps

```bash
# 1. Generate required files (from web/ directory)
nix develop -c go run ../pkg/hicli/cmdspec/print src/api/types/stdcommands.json src/api/types/stdcommands.d.ts src/api/types/commandtestdata

# 2. Create placeholder dist/ so the Go embed directive succeeds
mkdir -p web/dist && echo '<!-- dev -->' > web/dist/index.html

# 3. Build the Go backend
nix develop -c go build -o .dev-data/gomuks ./cmd/gomuks/

# 4. Install npm deps (from web/ directory)
nix shell nixpkgs#nodejs -c npm install

# 5. Create dev data directory structure
mkdir -p .dev-data/{config,data,cache,logs}
```

### Dev Config

Create `.dev-data/config/config.yaml` with auth disabled for local dev:

```yaml
web:
    listen_address: localhost:29325
    username: dev
    password_hash: $2a$12$anything
    token_key: devtokenkey1234567890abcdefghijklmnopqrstuvwxyz1234567890abcdefgh
    debug_endpoints: true
    event_buffer_size: 512
    origin_patterns:
        - "localhost:*"
        - "*.localhost:*"
    insecure_cookies: true
    disable_auth_because_i_want_my_account_to_be_hacked: true
logging:
    writers:
        - type: stdout
          format: pretty-colored
    min_level: debug
```

Key settings:
- `insecure_cookies: true` — required because Vite serves over HTTP, not HTTPS
- `disable_auth_because_i_want_my_account_to_be_hacked: true` — skips Basic Auth (the `fetch()` auth flow doesn't work well through the Vite proxy in dev)

### Running

```bash
# Start Go backend (separate data dir from production)
GOMUKS_ROOT=.dev-data nohup ./.dev-data/gomuks > .dev-data/gomuks-stdout.log 2>&1 &

# Start Vite dev server (from web/ directory)
nohup nix shell nixpkgs#nodejs -c npm run dev > .dev-data/vite-stdout.log 2>&1 &

# Launch browser with CDP remote debugging (temp profile)
nohup brave --remote-debugging-port=9222 --user-data-dir=$(mktemp -d /tmp/brave-debug.XXXXXX) --no-first-run --no-default-browser-check http://localhost:5173/ > .dev-data/brave-stdout.log 2>&1 &
```

Logs are at `.dev-data/gomuks-stdout.log`, `.dev-data/vite-stdout.log`, `.dev-data/brave-stdout.log`.

### CDP Remote Debugging

Verify CDP is running:
```bash
curl -s http://localhost:9222/json/version   # browser info
curl -s http://localhost:9222/json            # list open tabs
```

Execute JS in the browser via CDP (requires `python3` with `websockets`):
```bash
nix-shell -p 'python312.withPackages(ps: [ps.websockets])' --run 'python3 << "PYEOF"
import json, asyncio, websockets

async def main():
    # Get tab websocket URL from: curl -s http://localhost:9222/json
    uri = "ws://localhost:9222/devtools/page/<TAB_ID>"
    async with websockets.connect(uri, max_size=10*1024*1024) as ws:
        await ws.send(json.dumps({"id":1,"method":"Runtime.evaluate","params":{
            "expression": "JSON.stringify(window.client.store.preferences)"
        }}))
        resp = json.loads(await ws.recv())
        print(resp["result"]["result"]["value"])

asyncio.run(main())
PYEOF
'
```

Useful JS expressions for debugging state:
```js
// Check preferences
window.client.store.preferences.pin_favorites

// Inspect room list
window.client.store.roomList.current.length

// Find favorite rooms and their positions
window.client.store.roomList.current
    .map((r, i) => ({idx: i, name: r.name, fav: r.favorite_order}))
    .filter(r => r.fav !== undefined)

// Check a room's account data
window.client.store.rooms.get("!roomid:server").accountData.get("m.tag")

// Check connection state
window.client.rpc.connect.current

// Reload page via CDP
// Send: {"id":1,"method":"Page.reload","params":{"ignoreCache":true}}
```

### Notes

- `.dev-data/` is gitignored
- The dev instance uses a separate database — you need to log in with Matrix credentials on first use
- React `StrictMode` (enabled in dev via `web/src/main.tsx`) double-fires effects, which can cause spurious AbortErrors during auth
- Vite HMR picks up source changes automatically — no need to rebuild for frontend changes
- Go backend changes require rebuilding the binary and restarting
