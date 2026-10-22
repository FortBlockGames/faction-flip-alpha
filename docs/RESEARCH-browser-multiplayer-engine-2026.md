# Browser-based competitive 3D multiplayer with AI-assisted, community-driven iteration

Deep-research report, 22 Oct 2026. 23 sources fetched, 54 claims extracted, 25 verified by 3-vote adversarial review (19 confirmed, 6 refuted). Confidence labels are per finding.

## Recommendation

**Stay web-native. Build on three.js (what `faction-flip-alpha` already uses) or Babylon.js, in TypeScript with Vite, with Colyseus as the authoritative server, glTF/GLB assets from Blender, and all gameplay content as text/JSON in git.**

Why this over an editor engine:

1. **It is the stack LLM agents work best in.** Every verified agent-driven game framework (Godogen, 3AGameFactory) targets three.js/Babylon.js, serves the game at a live URL, and lets the agent judge results from the running game via browser capture. Scenes, content and assets are plain text, JSON and GLB. Claude Code can edit a weapon stat, a map layout or a skin manifest directly and open a PR. Binary Unity scenes/prefabs and Godot's editor-centric workflow are harder for agents.
2. **A patch is a redeploy, not an engine rebuild.** Static client on Vercel/CDN, Colyseus rooms on a Node host. Content JSON and GLBs ship via the same deploy.
3. **One-process dev loop.** The Colyseus Vite plugin (0.18.x) runs the server inside Vite: hot reload covers client code and server room definitions, active rooms are cached/disposed/restored with state and seat reservations preserved, clients auto-reconnect. Local dev only, but it makes "tweak a weapon, see it in a live match" a sub-second loop.
4. **Community input maps onto git.** Proposals become issues; approved ones become PRs that an agent implements against JSON content definitions; CI runs tests and deploys. Your Discord/Supabase quest platform (`conquest-platform`) is the natural front end for proposing and voting.

Fallbacks: **Unity** if higher-end visuals or its proven Addressables hot-content system outweigh agent-friendliness. **Godot 4** is a weaker web fit today (see below). **Unreal** has no credible browser path in the verified evidence.

## Engine comparison (fit to your constraints)

| Engine | Web renderer | WebGPU | Netcode on web | Hot content | Agent-friendliness | Verdict |
|---|---|---|---|---|---|---|
| three.js / Babylon.js + TS | WebGL2, WebGPU | Yes (three.js WebGPURenderer, Babylon native; fall back to WebGL2) | Any JS: Colyseus, raw WS, WebRTC, WebTransport | JSON + GLB, redeploy | Highest: all text, Godogen/3AGameFactory target it | **Recommended** |
| Godot 4.x | WebGL2 only (Compatibility renderer); no Forward+/Mobile | No upstream (proposal #6646 open) | WebSocket client + WebRTC only; browser cannot host | Resource packs possible, GDScript only | Good (.tscn is text) but web limits bite | Weaker for web today |
| Unity 6.3 LTS | WebGL2 default | Experimental in 6.3 LTS; label dropped in 6.6 (non-LTS, Sept 2026) | Netcode for GameObjects, Photon Fusion/Quantum (WebGL) | Addressables remote catalogs + differential content updates (assets only, no C#) | Lower: binary scenes/prefabs | Fallback for fidelity |
| Unreal 5 | No standard web export | n/a | n/a | ALIS plugin shows JSON-driven DataAssets, single-author | Low | Not viable for browser |
| PlayCanvas, Needle, Wonderland, Bevy-wasm, R3F | Not evaluated | | | | | The one comparison source for these was refuted 0-3; unevaluated |

## Verified findings

### Godot 4 web export (high confidence, 3-0 each)
- Browser builds are **WebGL 2.0 only** via the Compatibility renderer; Forward+/Mobile and WebGPU are unavailable through 4.5/4.6/master. Safari has WebGL2 issues; Chromium/Firefox recommended. WebGPU forks exist but are not upstream.
- **C# projects cannot be exported to the web.** GDScript (plus wasm GDExtensions) only. Still true in 4.7 stable; 4.8 is feature-frozen without it (PR #106125 is a WIP draft).
- Since 4.3, **single-threaded export is the default**; a plain HTTPS host works. Multi-threaded builds need SharedArrayBuffer and COOP/COEP headers, which force cross-origin isolation (no ads, no third-party embeds).
- **No UDP/ENet in the browser.** Only HTTP, WebSocket (client) and WebRTC. A browser build cannot host; you need a separate server or WebRTC signaling. Inactive tabs are paused and can disconnect networked sessions.

### Netcode (high confidence unless noted)
- **Colyseus** (Node/TypeScript rooms): WebSocket transports are production-grade. The WebTransport/QUIC transport (`@colyseus/h3-transport`, 0.18.4 published 2026-09-23) is officially experimental and "not battle tested". It is the only Colyseus transport with a true unreliable channel; on WebSocket, `.unreliable()` schema fields freeze after initial sync and unreliable input falls back to reliable. Secondary sources say Safari 26.4 shipped WebTransport in March 2026, so browser coverage is broader than the docs state.
- **Colyseus Vite plugin**: one `vite` process serves client and hosts server; HMR on both; rooms survive server-file saves. Reload is a whole-server-entry re-import, and a 13 Oct 2026 fix addressed PGlite corruption on reload, so it is new. Dev only.
- **Photon Quantum** (medium confidence): deterministic input-sync prediction/rollback, WebGL-capable since Quantum 3, lower bandwidth, steeper learning curve than Fusion. Claims that Photon recommends Fusion Shared Mode as the WebGL default, or that Quantum is CPU-unsuitable for WebGL, were refuted.
- **Pragmatic default for a first competitive title**: server-authoritative Colyseus room with client-side prediction and reconciliation. Rollback is a later upgrade if the genre demands it.
- Note: a secondary source reports **Hathora shutting down after May 2026**. Unverified, but treat Hathora as risky to depend on and confirm before choosing it.

### AI in the development loop (high confidence)
- **Godogen** (MIT, ~7.2k stars, updated 2026-10-16): generates a repo for Godot 4, Bevy, Babylon.js or three.js in which Claude Code or Codex builds the game, generates assets, runs it and judges success from the running game (live URL or 15-20 s proof recording), not from a clean compile. Browser capture uses Chromium with hardware WebGL2.
- **3AGameFactory** (OpenDCAI, Apache-2.0, ~938 stars, updated 2026-10-15): a coding agent (Claude Code, Codex, Gemini CLI) reads Skills and calls pipelines to turn a requirement into AI-generated 3D assets plus engine-ready code for UE5, Unity, Blender, Godot 4 and three.js (three.js r185 + Vite baseline). Asset pipeline: Meshy/Tripo/Trellis2 image-to-3D or text-to-3D, download GLB, automated mesh cleanup, visual QA over five orthographic views, polygon-budget check, scale validation, correction loop capped at three attempts.
- Caveat: this evidence shows what these tools target, not output quality. Third-party coverage notes weaker models need more guidance.

### Hot-updating content (high confidence)
- **Unity Addressables**: with Build Remote Catalog enabled before ship, clients resolve assets from a CDN catalog and fetch new content without reinstalling; differential content-update builds mean installed clients download only changed bundles. Limits: assets only (no C#), per-bundle granularity, load path fixed after ship, and on WebGL Unity's Caching API is unavailable so the browser cache stands in (reported re-download and stale-catalog issues).
- **Web-native stack**: equivalent is trivial. Content lives in JSON (weapons, balance, map manifests, skin registries) and GLB files; a deploy is a static push. No special system needed.
- **Unreal** (medium confidence): ALIS plugin suite regenerates typed DataAssets from JSON and propagates into placed actors without C++ recompile, but new field types still need C++, it is single-author with no independent users.

## Proposed architecture for TAKEOVER

- **Client**: three.js (or Babylon.js) + TypeScript + Vite. WebGPU renderer with WebGL2 fallback. Move `faction-flip-alpha` from a single 8.5 MB `index.html` to a Vite project so agents and contributors can work file-by-file.
- **Server**: Colyseus rooms (authoritative sim, client prediction). WebSocket in production now; WebTransport behind a flag. Host on Railway/Fly/Colyseus Cloud.
- **Content as data**: `content/weapons/*.json`, `content/maps/*.json` + GLB, `content/characters/*.json`, `content/skins/*.json`, `content/balance.json`. Validate with a JSON schema in CI. Shared by client and server so one PR changes both.
- **Assets**: Blender export to glTF/GLB (Draco or meshopt compression). Optional AI-generated assets via Meshy/Tripo with an automated QA pass modelled on 3AGameFactory's skill.
- **Community loop**: proposal in Discord or on the conquest platform (already Supabase + Discord bot) -> vote -> GitHub issue with a template -> Claude Code implements against the content JSON and opens a PR -> preview deploy on a Vercel branch URL + a dev Colyseus room -> playtest -> merge -> production. Gate with canon/IP review and an anti-exploit check before merge.
- **CI/CD**: lint, typecheck, Vitest, schema validation, headless Playwright smoke test (load, join room, fire weapon), then static deploy + server deploy. Feature flags in content JSON for staged rollouts and quick reverts.

## Open questions the research did not settle

1. Measured rendering quality, bundle size and load time for three.js vs Babylon.js vs PlayCanvas vs Unity WebGL at shooter-scale scenes, and whether WebGPU is stable enough to default on in 2026 browsers.
2. Which community-input mechanism (GitHub issues/PRs, in-game voting over a JSON registry, Discord bots) has been shown to work for live balance and cosmetics, and how to gate moderation and anti-exploit review on agent-implemented content.
3. Hosting comparison (Hathora status, PlayroomKit, Nakama, Colyseus Cloud) on cost, regions, tick-rate ceilings, WebTransport/WebRTC support; anti-cheat for fully inspectable browser clients.
4. When Godot web export gains WebGPU and C# (PR #106125, proposal #6646), which could change the picture for an editor-first team.

## Caveats
- Several primary docs sites (Godot, Unity, Colyseus, Photon, Meshy, Unreal forums) were blocked by the sandbox proxy; verifiers used the upstream GitHub doc repos, npm package source, or search-index extracts. Re-check wording before quoting Unity 6.3 WebGPU pages or the Photon blog verbatim.
- The engine ranking rests on constraint fit, not benchmarks. PlayCanvas, Needle, Wonderland, Bevy-wasm and React Three Fiber were effectively unevaluated after their comparison source was refuted.
- Time-sensitive: Godot WebGPU/C# web export, Colyseus WebTransport maturity, Unity 6.6 WebGPU status.

## Sources (primary and verified)
- https://docs.godotengine.org/de/4.5/tutorials/export/exporting_for_web.html and github.com/godotengine/godot-docs (4.2, 4.5, 4.6, master)
- https://github.com/godotengine/godot/pull/106125 ; https://github.com/godotengine/godot/issues/70796 ; https://github.com/godotengine/godot-proposals/issues/6646
- https://godotengine.org/article/progress-report-web-export-in-4-3
- https://docs.colyseus.io/server/transport/webtransport ; https://colyseus.io/blog/vite-integration/ ; github.com/colyseus/colyseus and github.com/colyseus/docs
- https://docs.unity3d.com/Packages/com.unity.addressables@2.7/manual/remote-content-intro.html ; github.com/needle-mirror/com.unity.addressables
- https://docs.unity3d.com/6000.3/Documentation/Manual/WebGPU-limitations.html ; https://discussions.unity.com/t/webgpu-out-of-experimental-in-unity-6-6/1734694
- https://github.com/htdt/godogen
- https://github.com/OpenDCAI/GameFactory-3A ; https://www.meshy.ai/blog/ai-agent-game-asset-pipeline
- https://blog.photonengine.com/photon-multiplayer-webgl-for-game-jams/ (snippet only)
- https://github.com/fallintodusk/alis ; https://forums.unrealengine.com/t/free-agpl-ue5-content-automation-json-dataassets-live-editor-updates/2733206
- Refuted and not to be cited: Viverse 2026 engine guide claims (0-3), Photon Fusion-Shared-Mode-default (1-2), Quantum CPU-load warning (0-3).
