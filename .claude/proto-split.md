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
- Versioning (owner, still being settled): MAJOR = breaking change to an existing message or service,
  which also bumps the package `vN`; MINOR = additions only.

## Open questions
- README versioning section needs rewriting once the owner settles the compatibility rules (see Decisions).
- Whether to start at v0.x while the API is still changing (package stays `v1`).

## Next session: start here
1. README fixes still pending from the second review:
   - Example command: the plugin must be named `protoc-gen-ts_proto`, and the files must be passed as `-I proto proto/*.proto`. Both were verified.
   - Lines 26-27: replace the "no tags/changelog" text, which stops mid-sentence, with the tag/release flow and versioning rules.
   - npm packages: `@bufbuild/protobuf`, `@grpc/grpc-js` and `ts-proto` (dev).
   - Link anchor `#usage-` should be `#usage`; the `.gitkeep` advice could be `mkdir -p` or a `!.gitkeep` rule instead; remove trailing spaces.
2. Settle the versioning rules and whether to start at v0.x.
3. Release workflow (step 17), then the first tag (step 16).
4. Api repo: `proto-gen-protoc.sh` must compile all files (step 12). That work belongs to the api repo.

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
11. [ ] Remove `service.proto.old` (owner).
12. [ ] Tell the api repo that `scripts/proto-gen-protoc.sh` must compile all files, not only the first.

## Checklist: release packaging
13. [x] `.gitattributes` with `export-ignore` for `CLAUDE.md`, `.claude/`, `.github/`, `.gitattributes`.
        This keeps them out of GitHub's tag archives, so a release archive holds only the protos + README.
14. [ ] Check locally: `git archive HEAD | tar t` should list only the files meant to ship.
15. [-] ~~`CHANGELOG.md`~~: dropped; the notes come from the tag.
16. [ ] First tag (`v1.0.0` or `v0.1.0`, still open) as an annotated tag; its message becomes the release notes.
17. [ ] `.github/workflows/release.yml`, triggered on `push: tags: ['v*']`: install protoc (`apt-get install protobuf-compiler`)
        -> syntax check -> `git archive --format=zip|tar.gz --prefix=sharkbot-proto/` ->
        `gh release create $TAG <files> --notes-from-tag` (needs `permissions: contents: write`).
        Still open: whether the owner writes it (Claude reviews) or Claude drafts it (logged here).
18. [ ] README: compile commands + how consumers fetch a tag (see below).

Consumer fetch:
- Public repo: `curl -sL https://github.com/project-sharkbot/proto/archive/refs/tags/v1.0.0.tar.gz | tar xz`
- Private repo: `gh release download v1.0.0 --repo project-sharkbot/proto --archive=tar.gz`

## TODO later
- README: a Windows section (installing protoc; `*.proto` globs are not expanded by cmd/PowerShell for protoc).
