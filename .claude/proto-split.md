# Proto split (Phase 1, item 2)

## Contribution log
- 2026-09-28: Claude wrote CLAUDE.md, reviewed service.proto/common.proto, and wrote the
  checklist below. The owner makes all proto edits themselves; Claude reviews them with protoc.
- 2026-09-28: the owner did the split (service.proto renamed to service.proto.old). Claude reviewed it:
  all files compile, every field and RPC from the old file is accounted for, and ts-proto codegen
  (the api's plugin, output written to the scratchpad) produces routes like `/aura.sharkbot.v1.EconomyService/AddNewUser`.
  Review notes are under Open questions.
- 2026-09-28: re-review after the owner's fixes. Renumbering, the PayUserRequest comment and the .idea/
  ignore are done. Tested `.gitattributes` in a scratch repo: `git archive` leaves out CLAUDE.md, .claude/,
  .github/, .gitattributes and README.md; it keeps .gitignore and the protos.
- 2026-09-28: third check. All protos compile; GameService duplicates removed; RulesetOverride renamed;
  README now ships and .gitignore is export-ignored. Claude reviewed the owner's README draft (the
  review comments were given in chat; the owner writes the text).
- 2026-09-28: second README review. Ran the README's TypeScript example in a mock project in the scratchpad:
  it fails twice (plugin named `protoc-gen-ts-proto` instead of `protoc-gen-ts_proto`; `*.proto` glob is
  expanded in cwd, not in the -I dir). It works with `-I proto proto/*.proto`. The generated code imports
  `@bufbuild/protobuf/wire` and `@grpc/grpc-js`.
- 2026-10-06: Claude explained the package version (major only, AIP-185/Buf) vs the release tag, what a
  patch means for a proto repo, and the SemVer 0.x caveat. The owner chose the tags and rewrote the README
  versioning section; Claude reviewed its wording (comments given in chat).
- 2026-10-06: second README review. Most points fixed (SemVer link, field numbers, npm packages, mkdir, Releases
  section). Still open: 0.x rule, `git push origin <tag>`, small grammar points, trailing spaces.
- 2026-10-06: third README review: 0.x section and push command fixed; only line 46 typo + trailing spaces left.
  Owner will write the release workflow; Claude gave guidance (structure, gotchas) in chat, will review it.
- 2026-10-06: reviewed the owner's draft release.yml (incomplete). Pointed out: sudo + apt-get update, stray
  quote in the protoc step, TAG/GH_TOKEN env missing, refetch of the annotated tag. Gave building blocks for
  the owner's tag check (annotated + non-empty message, version format with specific errors).
- 2026-10-06: second workflow review. Extracted the owner's check scripts and ran them in a scratch repo against
  test tags. Found: the loop includes `error`, so every valid tag fails; `$ {{` typo leaves GH_TOKEN literal;
  the package regex rejects `aura.sharkbot.v1.<domain>`; apt-get update missing; `format_failed` unused.
- 2026-10-06: third workflow review. All bugs fixed by the owner. Reran the checks in the scratch repo: valid
  tags pass (v0.1.0, v1.0.0); v0.1, v1.x.0, v01.2.0, v1.2.3.4, lightweight and empty-message tags fail with the
  right errors; naming accepts v1 for v1.0.0, rejects v2.0.0 and a `v10` package. Not testable locally: the
  tag refetch, apt install and `gh release create` (first real run will show). Commits proposed in chat.
- 2026-10-06: the owner committed and pushed `v0.1.0`; the release run succeeded. Claude checked the release
  with gh: notes equal the tag message, no extra assets. Downloaded both auto archives with
  `gh release download`: zip and tar.gz hold only README + 5 protos under `proto-0.1.0/`, and protoc
  compiles them. The curl URL fails (private repo). Updated Open questions and Next session.
- 2026-10-06: with the owner's go-ahead, Claude updated CLAUDE.md: the release process description (tag checks,
  auto archives, private repo, versioning summary) and the roadmap item (release done, `v0.1.0`).

## Decisions
- Errors use standard gRPC status codes; `RpcResult` is removed everywhere (owner, before this session).
- Packages follow the `aura.sharkbot.v1.<domain>` scheme already used in `common.proto` (owner).
- Files get domain names: common / moderation / economy / game (owner).
- Layout (owner): the domain files hold messages only; all three services (Moderation, Economy, Game)
  go in `services.proto` with `package aura.sharkbot.v1;`, so routes look like
  `/aura.sharkbot.v1.ModerationService/CheckMessage`.
- `CallerContext`: the `common.proto` version (`*_fk` fields, no `auth_metadata`); all platform IDs use the `*_fk` naming (owner).
- `GameService` no longer has `GenerateConnectionCode`/`AddConnection`; linking accounts is economy only (owner).
- Releases (owner): pushing a `v*` git tag runs a pipeline that does the protoc syntax check and, if it
  passes, creates a GitHub release with the protos as .zip and .tar.gz. No CHANGELOG.md file is kept;
  the change notes come from the annotated tag message or are auto-generated.
- Versioning (owner, 2026-10-06): SemVer tags `vM.m.p`, first tag `v0.1.0`. MAJOR = breaking change
  (also bumps the package `vN`); MINOR = additions; PATCH kept for SemVer tooling, only for changes with
  no wire/API effect (e.g. comments). Package stays `v1` during 0.x; before 1.0 breaking changes bump
  the minor (stated in the README).

## Open questions
- Pin format for consumers (how api/bots record which proto tag they use) and the rules in the owner's
  `version-check.sh`. During 0.x a consumer should pin an exact minor, since the minor bump can be breaking.
- Docker builds fetching protos from the private repo need a token in the build (e.g. a build secret for
  `gh`); decide when the api Dockerfile is written. `curl` on the archive URL only works if the repo is public.
- Mark 0.x releases as GitHub pre-releases (`gh release create --prerelease`)? Currently v0.1.0 is "Latest".
- Own release assets (stable checksums) vs GitHub's auto source archives (used now). Fine for Phase 1.

## Next session: start here
Proto split and release pipeline are done; `v0.1.0` is released (run 37525736249, success).
1. Api repo (separate session, owner): `scripts/proto-gen-protoc.sh` must compile all files, not only
   the first (step 12), and fetch a pinned tag. Archives unpack to `proto-<version>/` (e.g. `proto-0.1.0/`),
   so the `-I` path must match.
2. Optional: a `ci.yml` running the protoc syntax check on push/PR to `main`, so errors show before tagging.
3. Next proto changes follow the README rules: during 0.x every release bumps the minor; after `v1.0.0`
   never renumber or reuse field numbers (use `reserved`).
4. TODO later items below.

## Checklist: split
1. [x] `common.proto`: keep the 5 enums and `CallerContext`. Settle the `CallerContext` field names.
2. [x] `moderation.proto`: `package aura.sharkbot.v1.moderation;`, `import "common.proto";`,
       timestamp + struct imports. Move all moderation messages here, including `RuleConfig`,
       `Ruleset` and `RulesetOverride`. No `service` block.
3. [x] `economy.proto`: `package aura.sharkbot.v1.economy;`, `import "common.proto";`, timestamp + struct.
       Move all economy messages here, including `TransactionRecord`. No `service` block.
4. [x] `game.proto` (new): `package aura.sharkbot.v1.game;`, `import "common.proto";`, struct.
       Move all game messages here. No `service` block.
5. [x] `services.proto`: `package aura.sharkbot.v1;`; import common, moderation, economy and game
       (no well-known-type imports needed). Add `ModerationService`, `EconomyService` and `GameService`,
       with qualified types, e.g. `rpc CheckMessage (moderation.CheckMessageRequest) returns (moderation.CheckMessageResponse);`.
6. [x] Delete every `RpcResult result = 1;` (8 in moderation, 7 in economy, 1 in `CreateGameResponse`)
       and shift the remaining field numbers down by 1. This is safe only before `v1.0.0` is tagged;
       after that, never renumber or reuse numbers (use `reserved`).
7. [x] In the domain files, qualify shared types as `common.CallerContext`, `common.Platform`, etc.
       (`common.X` resolves outward to `aura.sharkbot.v1.common.X`).
8. [x] Fix comments that mention `RpcResult` (for example the `GameService` header comment).
9. [x] Check: `protoc -I . --descriptor_set_out=/dev/null *.proto` passes with no output.
10. [x] Check that `grep -n RpcResult *.proto` returns nothing outside `service.proto`.
11. [x] Remove `service.proto.old` (owner).
12. [ ] Tell the api repo that `scripts/proto-gen-protoc.sh` must compile all files, not only the first.

## Checklist: release packaging
13. [x] `.gitattributes` with `export-ignore` for `CLAUDE.md`, `.claude/`, `.github/`, `.gitattributes`.
        This keeps them out of GitHub's tag archives, so a release archive holds only the protos + README.
14. [x] Check locally: `git archive HEAD | tar t` should list only the files meant to ship.
15. [-] ~~`CHANGELOG.md`~~: dropped; the notes come from the tag.
16. [x] First tag `v0.1.0` (annotated); its message became the release notes.
17. [x] `.github/workflows/release.yml`, triggered on `push: tags: ['v*']`: install protoc (`apt-get install protobuf-compiler`)
        -> syntax check -> `git archive --format=zip|tar.gz --prefix=sharkbot-proto/` ->
        `gh release create $TAG <files> --notes-from-tag` (needs `permissions: contents: write`).
        Owner writes it, Claude reviews. Gotchas given: annotated tag lost by actions/checkout, GH_TOKEN for gh,
        libprotobuf-dev for well-known types, GitHub's own source archives already honor export-ignore.
18. [x] README: compile commands + how consumers fetch a tag.

Consumer fetch:
- Public repo: `curl -sL https://github.com/project-sharkbot/proto/archive/refs/tags/v0.1.0.tar.gz | tar xz`
  (fails now: the repo is private).
- Private repo: `gh release download v0.1.0 --repo project-sharkbot/proto --archive=tar.gz` (tested 2026-10-06).

## TODO later
- README: a Windows section (installing protoc; `*.proto` globs are not expanded by cmd/PowerShell for protoc).
