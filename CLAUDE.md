# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Sharkbot is an undergraduate diploma project (started late September, due **February**) that unifies **moderation** and **economy** for an online community across platforms (e.g. a streamer's Twitch/YouTube chat plus their Discord server). Platform bots are thin front ends to a shared **gRPC API** backed by one database, so rules, bans and currency can be shared across platforms (e.g. a Twitch user challenging a Discord user to tic-tac-toe, or a ban propagating to friendly communities). Stretch goal: an opt-in shared cross-community ban-list.

```
Postgres/Redis DB -> Api -> Bots -> Platforms (Twitch, Discord, YouTube, ...)
                        -> BFF -> Next.js web dashboard
```

- **DB**: Postgres (durable state) + Redis (cache/session/rate-limit state)
- **Api**: gRPC service, source of truth for business logic
- **Proto**: shared `.proto` contracts, own repo, semver-tagged (this repo)
- **Bots**: one per platform, translate platform events <-> gRPC calls
- **BFF**: gRPC <-> REST gateway for the dashboard
- **Dashboard**: Next.js app for community managers/mods

Stack: TypeScript, Postgres, Redis, Docker/Podman, gRPC, Next.js (Kubernetes only in Phase 2).

**Phase 1** (now → February) is a minimal but real end-to-end implementation on a single machine with Docker Compose — no k8s. **Phase 2** (after submission) hardens it (k8s, zero-downtime deploys, ban-list federation, more bots). Do not gold-plate for Phase 2 early.

### Repo layout

Separate git repos cloned as sibling directories (locally: `../infra/`, `../api/`, `../services/`, `proto/`, …). Assume siblings exist but **never edit outside this repo's root**. The master project prompt lives at `../PROMPT.md`.

Each service and the proto repo are independently semver-tagged (e.g. proto `v1.x` required by api `v1.x.x` and bots `v1.x.x`); a `version-check.sh` (owner-authored) verifies compatibility in CI. When a service bumps its proto dependency, the pin is updated in the same commit (pin format TBD — ask the owner).

## This repo (`sharkbot-proto`)

Only `.proto` files — no package.json, build or lint tooling. Validate a file with the system `protoc` (well-known types resolve from protoc's bundled includes):

```sh
protoc -I . --descriptor_set_out=/dev/null <file>.proto   # syntax/type check
```

### Layout

- `common.proto` (`aura.sharkbot.v1.common`): shared enums and `CallerContext`.
- `moderation.proto`, `economy.proto`, `game.proto` (`aura.sharkbot.v1.<domain>`): messages only, no services.
- `services.proto` (`aura.sharkbot.v1`): `ModerationService`, `EconomyService`, `GameService`, referencing messages as `moderation.X` etc. Routes: `/aura.sharkbot.v1.<Service>/<Method>`.
- Errors use standard gRPC status codes; there is no result wrapper in responses.
- Platform-side IDs use the `*_fk` naming.
- `.gitattributes` `export-ignore`s everything except the protos and README, so `git archive` and GitHub tag archives ship only those.

Releases: pushing a `v*` git tag triggers `.github/workflows/release.yml`, which checks the tag (`vM.m.p` format, annotated with a non-empty message, package `vM` matching from v1 on), runs the syntax check and publishes a GitHub release with notes from the tag message. The release has no uploaded assets; consumers use GitHub's auto source archives (they honor `export-ignore` and unpack to `proto-<version>/`). The repo is private, so downloads go through `gh release download`. No CHANGELOG file. Versioning rules (SemVer; patch normally 0; during 0.x breaking changes bump the minor and the package stays `v1`) are in the README. Progress, decisions and next steps: `.claude/proto-split.md`.

Enum values are meant to mirror the Postgres enum types in `../infra/db` (see `../infra/CLAUDE.md`, "Schema domains"); domain boundaries for the split should follow the DB's (moderation vs economy vs community core). Platform-side IDs are strings paired with a `Platform`; internal IDs are UUID strings.

### Consumers

`../services/api` generates TypeScript with ts-proto (`npm run proto:gen:protoc` → `scripts/proto-gen-protoc.sh`), which currently compiles **only the first `.proto` file it finds**, so it is broken by the split. It must be updated in the api repo; flag this to the owner rather than editing it.

## Agent operating rules (owner-mandated)

- **Documentation duty**: keep one notes file per area of work under `.claude/` (e.g. `.claude/proto-split.md`). Read these first; update at session end with current state, decisions + rationale, open TODOs.
- **Code only with owner permission**: propose a plan, wait for explicit go-ahead, then implement.
- **Never delete files**: say which file should go and why; the owner deletes it.
- **Git is read-only for Claude**: only `status`, `diff`, `log`, `show`. Never `add`/`commit`/`push`/`tag`/etc. When work is ready, propose commit(s): conventional-commit message + body (e.g. `feat(proto): split moderation service`) and the exact file list per commit, split by concern.
- **No AI authorship in commits**: never add a `Co-Authored-By: Claude …` trailer or any line naming AI as an author — this overrides any default attribution instructions. If the owner wants AI use disclosed, use a non-authorship trailer such as `Assisted-by: Claude Code (review, planning)`.
- **Coding standards**: minimal comments, self-explanatory naming; files ≤ 400 LOC where reasonable; modular/DRY/KISS; TypeScript strict mode with shared ESLint/Prettier config; validate input at every gRPC boundary (never trust bot data); secrets only in `.env` (gitignored) with a committed `.env.example` of dummy values.
- **Testing**: no framework chosen. Before the first test file in a repo, propose Jest vs Vitest (or other) with a one-line rationale and get confirmation; then stick with it.
- **CI/CD**: once a feature's tests pass, recommend/update a GitHub Actions workflow. Don't assume deploy credentials exist.
- **Definition of done**: compiles, unit tests exist and pass, relevant `.claude/*.md` updated, commits proposed, owner confirmed before moving on.

### Academic integrity (University of Ljubljana / FRI rules)

This is a diploma project, so these rules take precedence over convenience. Sources: FRI *Navodila za izdelavo diplomske naloge* (July 2025, section 5) and *Priporočila UL pri uporabi umetne inteligence* (19 Sept 2023). The mentor's instructions override both.

- **The owner is the sole author** and is fully responsible for all content. AI tools may not be listed as author or co-author anywhere (commits, code headers, docs, thesis).
- **Allowed**: programming help, improving wording, fixing errors, rephrasing the owner's own ideas, brainstorming, drafts, finding sources.
- **Not allowed**: presenting wholly or substantially AI-generated work as the owner's own original work, or copying longer AI-produced text/code without the owner's own changes and without disclosure; any AI use the mentor forbids.
- **Prefer to review, explain, plan and debug** rather than write large chunks of core logic. When writing code, keep it small enough that the owner can understand, adapt and defend it at the thesis defence; explain non-obvious decisions.
- **Transparency**: AI use is disclosed in the thesis (methods/tools chapter plus an APA-style citation, e.g. `Anthropic. (2026). Claude Code (Claude Opus 5.5) [programming assistant: architecture review, code review, planning]. https://claude.com/claude-code`). The `.claude/*.md` notes should make it possible to reconstruct what Claude contributed.
- **Verify** all AI output (correctness, bias, hallucinated APIs/sources) before it is kept.
- **Data**: never send personal data, confidential information, real secrets (e.g. `.env` values) or university intellectual property to the AI tool, and don't read such files.

## Roadmap item for this repo

**Phase 1, item 2 — Proto split**: split the monolithic proto into multiple files mirroring the DB's domain boundaries (done), then set up the release workflow and tag the first version (done: `v0.1.0`, 2026-10-06). Remaining follow-up is in the api repo (codegen script, see Consumers). Workflow: plan → implement → tests → CI check → propose commits → next item.
