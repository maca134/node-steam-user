# CLAUDE.md

Our fork of `steam-user` (upstream: DoctorMcKay/node-steam-user, fork: maca134/node-steam-user). Lives as a submodule
at `libs/steam-user` in the dzsa-launcher repo but is worked on as its own repo.

## Why the fork exists

dzsal master list (`legacy/master-list/src/sources/client.ts` in the parent repo) uses it as a fallback DayZ server list
source: logs on with a refresh token, connects over TCP, sends a raw GMS query
(`client._send(6403, { app_id: 221100, filter_text, max_servers: 20000 }, cb)`). Master list is pinned to
`oven/bun:1.3.14-slim` because TCP logon gets no response on Bun 1.4.2/1.4.3. Plan and findings:
`docs/rewrite/steam-user-bun.md` in the parent repo.

## Rules

- Keep changes minimal and close to upstream so merging upstream stays easy. Don't reformat or refactor.
- Commit locally; never push (the user pushes). Note anything Bun-specific in the commit message.
- Anonymous logon only for tests. NEVER log on with prod's Steam account or refresh token: a second session kicks
  prod's client. Ask first if a real account is needed.
- Don't touch prod servers or the master-list repo from here.
- Follow existing style: tabs, `.editorconfig`, `.eslintrc.js` (`npm run lint`).
- Absolute paths in file operations. Be concise; don't overstate what works; say when something is untested.
- Ask before acting if the request is ambiguous. Don't change anything not asked for.
- Code comments only for things the next reader would spend several tool calls working out.
- Avoid shell commands: don't use `find` or `grep` in tests; use Bun's Glob and built-in tools.
- If you need a paragraph-long comment to justify why the workaround is OK, the code is wrong: fix the code.
- Preplan your tool calls, and group in batches where it makes sense.
- When reporting to the user, be extremely concise and sacrifice grammar for sake of concision.

## Layout

- `index.js`, `components/*.js`: mixins loaded in numeric order into `SteamUser`.
- `components/02-connection.js`: connect, reconnect, protocol choice.
- `components/connection_protocols/{base,tcp,websocket}.js`: transports. TCP framing and encryption handshake are in
  `tcp.js`; WebSocket uses `websocket13`.
- `components/09-logon.js`: logOn flow. `03-messages.js`: send/receive, multi-message (gzip) handling.
- `protobufs/`, `enums/`: generated (`npm run generate-protos`, `generate-enums`); don't hand-edit.

## Testing on Bun (Docker)

Docker Desktop is installed. Test across `oven/bun:1.3.14-slim`, `oven/bun:1.4.2-slim`, `oven/bun:1.4.3-slim`.
Results are flaky, so run each several times and record counts.

```
docker run --rm -v "<fork path>:/app" -v "<repro dir>:/repro" -w /app oven/bun:1.4.3-slim sh -c "bun install && bun /repro/logon.ts"
```

`logon.ts`: `new SteamUser({ protocol: SteamUser.EConnectionProtocol.TCP })`, `logOn({ anonymous: true })`, exit 0 on
`loggedOn`, 1 on `error`, 2 after 30s with no response. Add `client.on("debug", console.log)` to see where the
handshake stops. Keep repro scripts outside the repo (scratchpad). The GMS query (6403) gets no reply on anonymous
logon even on 1.3.14, so don't use it to judge a fix.

## Status

See the Findings section of `docs/rewrite/steam-user-bun.md` in the parent repo; update it as results come in.
