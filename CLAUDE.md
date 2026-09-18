# holaOS (holaboss-ai/holaOS)

## 프로젝트 개요
기업이 사내 보안 규정에 맞춰 완벽하게 커스텀하고 독립 호스팅할 수 있는 "엔터프라이즈 맞춤형 자율 에이전트 운영체제"
회사 기밀 데이터가 외부 클라우드로 새어 나가지 않는 철통 보안 속에서 다양한 부서의 맞춤형 AI 직원들을 한곳에서 운영
글로벌 대기업 수준의 첨단 에이전트 자동화 시스템을 사내 인프라에 안전하게 구축하고 싶은 엔터프라이즈의 표준 플랫폼

## 핵심 특징 & 추천 분야
- 기업맞춤에이전트OS
- 사내독립호스팅
- 기업기밀철통보안
- 부서별AI직원배치
- 엔터프라이즈자동화

---
*이 문서는 오픈소스 큐레이터(Curator-Agent)에 의해 자동 생성된 가이드 문서입니다.*


---
## 기존 CLAUDE.md 내용

# holaOS — repo guide

The local Electron desktop app (`hola-boss-oss`) + its in-process runtime
(`runtime/{harnesses,harness-host,api-server,state-store}`). See
`apps/desktop/CLAUDE.md` for desktop-renderer conventions and the parent
`~/Developer/CLAUDE.md` for how this fits the wider Holaboss product.

## `hola` — debug the Hola (pi) brain from the CLI

`scripts/hola.mts` runs the **pi (Hola)** harness **in-process from source** so you
can debug the brain without the desktop UI: breakpoints in `runtime/harness-host/src/pi.ts`,
edit-and-rerun with no build/stage loop, and spin up multiple instances.

```bash
# Close the desktop for THIS checkout first (see "why" below), then:
npm --prefix runtime/api-server run hola -- -p "list the files in this repo and summarize it"
```

How it works: it calls the runtime's real `executeTsRunnerRequest` pipeline and
overrides only the `runHarnessHost` dep to run `runPi()` in-process instead of
spawning `harness-host run-pi`. So every build stage (MCP, sidecar, skills,
tools, `model_client`, injected context) is **faithful** to a desktop run; only
the harness subprocess is swapped. Events stream through the real relay (so
`harness_session_id` is persisted → resume works) and are pretty-printed.

Key facts:
- **Root auto-detected** from `apps/desktop/.env` (`HOLABOSS_DESKTOP_USER_DATA_DIR`
  → `<appData>/<dir>/sandbox-host`, same as the desktop). Override with
  `--sandbox-root <path>` or `HB_SANDBOX_ROOT`.
- **Owns the root runtime** ("option 1"): run with the **desktop closed** on that
  root. The tool refuses a root whose `data.db-wal` is hot (a live desktop) unless
  `--force` — opening a live root's `data.db` risks write contention. Each `hola`
  run launches/reuses a runtime backend (for the HTTP-backed tools: runtime-agent
  tools / composio / browser / web-search); `--no-runtime` skips it (brain + model
  + workspace MCP still work).
- Creds: `model_proxy_api_key` is the **auth token**, `sandbox_id` from
  `resolveProductRuntimeConfig()` (reads the root's `state/runtime-config.json`) —
  so it reuses your desktop login; no separate auth.

Flags: `-p/--prompt`, `--cwd`, `-m/--model`, `-s/--session <path>` (resume a
specific session; default auto-resumes the workspace's last one), `--fresh`
(new session, won't overwrite the saved one), `--no-runtime`, `--keep` (leave the
launched runtime up), `--force`, `--print-request` (build + print, no model
call — still opens the DB), `--debug` (raw events), `--port`.

Implementation notes for future edits:
- The file is `.mts` (forces ESM; `ts-runner.ts` has a top-level await that breaks
  CJS transform) and lives in `scripts/` (outside every package's `tsconfig` so it
  never pollutes `tsc --noEmit`). Run via the api-server's `tsx`. `prehola` rebuilds
  better-sqlite3 for the current node ABI.
- `runPi(req, deps)` REPLACES its deps — pass `{ ...defaultPiDeps(), emitEvent }`,
  not just `{ emitEvent }`, or you'll hit `deps.createSession is not a function`.
- Apply the root env (`HB_SANDBOX_ROOT` + DB paths) BEFORE launching the runtime
  child, else it defaults to `/holaboss` and dies with `mkdir: /holaboss:
  Read-only file system`.
