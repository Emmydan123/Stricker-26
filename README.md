# Striker 26 — deploy + online play

Two pages, no build step, no framework, no CDN at runtime:

| File | What it is |
|---|---|
| **`index.html`** | The game. Menu → *VS CPU* / *2 PLAYERS* / *ONLINE*. All game code + the PeerJS WebRTC library are inlined, so this one file runs on its own. |
| **`multiplayer.html`** | The "join a match" page — reached at **`/multiplayer`**. The second player types the room code here; it hands off to `index.html?join=CODE`, which connects and starts the match. |
| `manifest.webmanifest`, `icon-*.png`, `apple-touch-icon.png` | Installable-app icon + full-screen launch. |
| `sw.js` | Service worker: caches both pages so the game is playable offline afterwards. |
| `netlify.toml` / `_headers` | `/multiplayer` → `multiplayer.html` rewrite, cache + security headers. |

---

## How online multiplayer works

1. Player A: opens the site → **ONLINE** → **CREATE ROOM**. A 5-letter code appears (e.g. `V9DZN`).
2. Player B: opens **`your-site.netlify.app/multiplayer`** → types `V9DZN` → **CONNECT**.
3. The match starts on both phones. The host is the **home** team, the joiner is **away**.
4. Each player gets the landscape layout: movement stick bottom-left, PASS / SHOOT / ⚡ bottom-right.

Under the hood: the two phones talk **directly peer-to-peer over WebRTC**. PeerJS is used only for the
introductions (the free PeerJS broker), then the ball never touches a server — which keeps it fast
(~10 ms of latency on typical mobile data on the same continent).

- The **host owns the simulation** and streams ~20 snapshots/second; the guest sends its inputs 30×/second.
- If the opponent drops, the host is warned and the computer takes over that team — the match keeps going.
- Guest leaving = connection closed. Host leaving = the guest shows "opponent left" and keeps spectating the result.
- Both players must be online *while playing online*. Everything else (vs CPU, 2 players on one phone) is offline.

### Limits worth knowing
- Peer-to-peer needs both phones on the internet, and it can be blocked by strict corporate/school Wi-Fi NATs. Mobile data and home Wi-Fi are fine.
- If the free PeerJS broker is ever down you can point the game at your own PeerServer without touching the game code: add one line before the game script loads —
  `<script>window.STRIKER_PEER_OPTS = { host: 'peer.example.com', port: 443, path: '/' };</script>`
- A room code is not password-protected, it is just a random unguessable ID.

---

## Deploying to Netlify

### Option 1 — drag & drop (30 s)
1. Go to **https://app.netlify.com/drop**
2. Drag the **whole folder** (or `striker26-site.zip`) onto the page.
3. You get a URL like `https://striker26-xyz.netlify.app`. Rename it under *Site configuration → Change site name*.
4. Phone: open the URL → browser menu → **Add to Home screen** for a full-screen icon.

### Option 2 — Git + auto-deploy (best if you'll edit)
```bash
git init && git add . && git commit -m "Striker 26"
git branch -M main
git remote add origin https://github.com/<you>/striker26.git
git push -u origin main
```
Netlify → *Add new site → Import an existing project → GitHub*. Build command: **empty**. Publish directory: **`.`**

### Option 3 — CLI
```bash
npm i -g netlify-cli
netlify login
netlify deploy --prod --dir .
```

`netlify.toml` already wires up the pretty URLs, so all of these work after deploy:

- `/` → game
- `/multiplayer` → join-with-code page
- `/join` → same page (alias)
- `/?join=V9DZN` → game that auto-connects to that room
- `/?host=1` → game that immediately creates a room

## Verifying it worked
- Open `/multiplayer` on a phone — you should see the code entry screen, in landscape.
- Create a room on one device, join it from another. Both should show the pitch within a second or two.
- Airplane mode after a first visit: the game still launches (online play obviously needs a connection).

Bump `CACHE = 'striker26-vN'` in `sw.js` whenever you want to force every returning player to pull the new build.
