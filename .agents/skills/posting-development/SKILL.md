---
name: posting-development
description: Use when adding, changing, debugging, or reviewing functionality in this Posting repository; includes its Go architecture, feature extension points, compatibility constraints, and verification workflow.
---

# Developing Posting

## Repository baseline

- This is Posting 3: a Go terminal API client. The Go module is `github.com/darrenburns/posting/v3`; `go.mod` requires Go 1.25.5. The primary application entry point is `cmd/posting`.
- The default development target is the Go code under `cmd/` and `internal/`. `src/posting/`, `pyproject.toml`, and the root `tests/*.py` are Posting 2's Python implementation and tests, retained for maintenance and CI. Do not implement a Posting 3 feature in Python unless the user specifically asks for Posting 2 or its maintenance branch.
- Go UI uses Terma (`github.com/darrenburns/terma`). Requests are stored as Posting 2-compatible `.posting.yaml` files where possible; Posting 3 adds kind-specific fields.
- The top-level `skills/using-posting/SKILL.md` is a different artifact: it is embedded in the binary and printed by `posting remote skill`. This development skill lives in `.agents/skills/` and is not embedded.
- Product docs can lag code: verify current behavior in implementation/tests and changelog, not only `docs/roadmap.md` (for example, its WebSocket roadmap entry is stale relative to implemented code).
## Local development installation

- Create a dedicated Git branch before changing Posting. Preserve existing user changes.
- After changing application code, build and replace the installed `posting` executable so it can be run globally without entering the repository. Find the active executable with `command -v posting` and replace that exact path using `go build -o "$(command -v posting)" ./cmd/posting` from the repository root; confirm `posting version` runs the new build. Do this for every subsequent application-code change.
- Do not rebuild for documentation-only or skill-only edits.

## Package map

| Path | Responsibility |
| --- | --- |
| `cmd/posting/` | CLI entry point, flags/subcommands, application composition and startup, headless `posting run`, import and remote commands. |
| `internal/model/` | Toolkit- and filesystem-independent request, response, request-kind, variable, and history data; request resolution and normalization. |
| `internal/ui/` | Terma app, request sessions/editors, panels, commands, response views, environment and collection interactions. |
| `internal/client/` | Sender boundary and HTTP, gRPC, WebSocket, SSE, OAuth and TLS transports. |
| `internal/collection/` | Read/write collection directories and `.posting.yaml` request serialization. |
| `internal/env/`, `internal/config/`, `internal/paths/` | Environment layering and substitution inputs, user settings, and platform-specific paths. |
| `internal/history/` | Persistent per-collection exchange history. |
| `internal/importing/` | OpenAPI, Postman and Bruno readers and collection-writing helpers. |
| `internal/remote/` | Local remote-control protocol and client/server; UI integration lives in `internal/ui/remote.go`. |
| `internal/themes/`, `internal/curl/`, `internal/grpcurl/` | Theme loading, curl conversion, and grpcurl formatting. |
| `skills/` | Skill distributed by the built Posting executable; not repository-development guidance. |
| `docs/guide/`, `docs/CHANGELOG.md` | User-facing docs and release history. |

## Data and request flow

The app composition root is `cmd/posting/main.go` (`appConfig` and `run`). It loads settings, collection, environments, theme and TLS options, then injects stores/watchers and request clients into `ui.Config`. Keep new dependencies or platform wiring at this boundary rather than hiding filesystem/network setup in a widget.

For an interactive request:

1. `internal/ui/session.go`: `Session.Snapshot` converts editor state to `model.Request`; `Session.Load` restores a model value.
2. `internal/ui/app.go`: `App.start` takes the snapshot, resolves request-scoped variables, creates `client.Call`, and passes it to `Session.Send`.
3. `internal/client/client.go`: `Sender` is the transport boundary. Implementations must honor context cancellation; progress callbacks may run off the UI goroutine.
4. `internal/ui/session.go`: completion is dispatched back to the UI goroutine; response state, capture variables and history are updated.

The UI is configured with injected senders, stores, watchers and environment sources; tests commonly use fake implementations. Preserve this boundary. The headless `posting run` path is separately composed in `cmd/posting/run.go`; changes to request execution or serialization may need to cover both interactive and headless callers. Remote commands enter through `internal/ui/remote.go` and use the configured UI app.

`model.Request` is the unresolved, user-authored request. Resolve variables at the existing request boundary; don't eagerly rewrite the saved/editor model. Use `model.Normalize` for kind-specific shared fields and `model.Resolve`/environment resolution patterns rather than duplicating substitution.

## If adding a request kind

A new kind is intentionally cross-cutting; do not stop after adding an enum. Trace and update every applicable layer:

1. **Model:** `internal/model/kind.go` (`KindID`, `Kind`, `Fields`, `Kinds`, `Payload`, `Normalize`, lowering/status behavior); add the payload type in `internal/model/`. Account for clone, resolution, status, size/history and sample request behavior.
2. **Collection format:** `internal/collection/format.go` and `internal/collection/kinds.go`: parse and marshal its YAML block, reject invalid/misplaced fields consistently, and preserve Posting 2 compatibility. Extend the all-kinds request file round-trip test.
3. **Transport:** `internal/client/` sender implementation and `internal/client/bykind.go`; wire it in both `cmd/posting/main.go` (`appConfig`) and `cmd/posting/run.go` where the feature supports headless sending.
4. **UI:** `internal/ui/kinds.go` `kindViews`, editor/payload `Snapshot`/`Load`, method/kind selector, applicable request and response tabs, actions, command palette, exports and response handling. Check `internal/ui/session.go`, `app.go`, `overlays.go`, `response_view.go` and existing kind-specific editors for the actual paths.
5. **Compatibility and docs:** inspect importers, curl conversion, history/remote JSON and `docs/guide/requests.md` where relevant; update the user guide and changelog for user-visible changes.

Existing completeness checks include loops over `model.Kinds`, collection round trips, transport dispatch checks and `internal/ui/kinds_test.go:TestEveryKindIsWired`. Use them; add only behavior-focused tests for gaps. Not every feature is a new request kind—reuse the existing model, editor, action and transport patterns that fit the requested behavior.

## Implementation conventions

- Read the affected implementation and its callers/tests before editing. Search for shared contracts and feature registries; avoid fixing only one caller when behavior is shared.
- Keep toolkit concerns in `internal/ui`, transport concerns in `internal/client`, and domain rules in `internal/model`. Use existing types and interfaces. Avoid speculative abstractions and new dependencies.
- UI session state is owned by the UI goroutine. For asynchronous work, follow `Session.Send`/`dispatch` patterns and honor cancellation/generation handling; do not mutate widget state directly from a worker goroutine.
- Collection files are user data. Preserve field omission/default behavior, strict validation, ordering where meaningful, round-trip fidelity, and old collection compatibility. Do not silently drop unsupported fields.
- Protect secrets and credentials in logs, display and persistence. Follow existing TLS/OAuth and environment rules for security-sensitive changes.
- Go tests are colocated as `*_test.go` and use the standard `testing` package. UI tests use the repository's Terma helpers; inspect nearby tests before adding snapshot coverage. Update snapshots only when the changed UI output is intentional, and inspect the diff first.
- User-visible Go changes belong in `docs/CHANGELOG.md` under `[Unreleased]`; if that section is absent, add it at the top as `CONTRIBUTING.md` specifies. Update the relevant `docs/guide/` page when behavior, configuration or usage changes. Do not modify docs or changelogs for internal-only changes with no user-facing impact.

## Development and verification

Run from the repository root with Go 1.25.5 or newer:

```sh
gofmt -w <changed-go-files>
go test ./internal/<affected-package> -count=1
go build ./...
go vet ./...
go test ./...
```

CI checks Go formatting in `cmd internal`, then `go build ./...`, `go vet ./...`, and `go test ./...` on Linux and macOS. Run the focused package test while iterating and the full relevant checks before delivery. For documentation changes, follow `CONTRIBUTING.md`: `uv run mkdocs build --strict`.

For interactive smoke testing, the repository provides sample data:

```sh
go run ./cmd/posting --collection tests/sample-collections/ \
  --env tests/sample-envs/sample_base.env \
  --env tests/sample-envs/sample_extra.env
```

Exercise the changed UI or request path in the running app when the change affects user-visible behavior. For transport behavior, prefer a local test server/fake and test the observable request/response semantics; don't make verification depend on external APIs or credentials. If only docs or non-runtime skill content changed, inspect the rendered/source content instead of running unrelated tests.

The Python tests are still part of CI, but are not the verification path for Go Posting 3 changes. For an explicitly requested Posting 2 change, follow the Python instructions in `CONTRIBUTING.md` instead.
