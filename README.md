# Sharkbot proto files

This repository contains the protobuf files for the Sharkbot project's gRPC communication and message definition. 

This README.md file is to explain the structure, usage and compilation of these files for your usage or for an agent to read and understand how to use.

## Structure

```text
├─ common.proto         // contains structures used in all the other proto files
├─ economy.proto        // structures for economy specific requirements
├─ game.proto           // the same but for games
├─ moderation.proto     // the same but for moderation
└─ services.proto       // defines rpc services for games, moderation and economy
```

All the files are required to compile protocol buffers for use in a project, as they import structures from each other.

### Naming/Versioning convention 

Packages are named like `owner.project.version.<module>` for example `common.proto` is `aura.sharkbot.v1.common` package. 
The exception here is `services.proto` which ends at version. The result is like `owner.project.version.Service/Method`.
For example: `/aura.sharkbot.v1.ModerationService/CheckMessage`.

Versioning follows Semantic versioning ([SemVer](https://semver.org/)) - `vM.m.p` where `M` indicates major (breaking) 
changes like renamed fields, changed types or changing field numbers. This number should also match the package name. 

Next, `m` marks minor changes like added fields, new enums, rpc methods or services. 
This kind of change should keep consumers built against an older minor working.

Lastly is `p` which is kept for compatibility with other version checking software but not necessarily required. 
The "patches" would be for example adding/changing comments, since protobuf generators can include them in the generated 
code. But overall such changes do not introduce any kind of practical alteration to the code or structure. 
It is for this reason that the patch number of any release should typically be 0.

#### Pre-release versions

For simplicity the `v0.x.x` tags' breaking changes will only bump the minor version. 
The package names will also be named `v1` until that release is ready. After `v1.0.0` tag is pushed, the rules above
will apply.

### Releases 

To create a new release, a new annotated tag should be created matching the versioning annotation. After the tag with
described changes is pushed (`git tag -a vM.m.p -m "change notes"` and `git push <remote> vM.m.p`), a GitHub workflow 
creates a release. Servers/clients would pull these releases during compilation (like building the docker image) or 
developers could download and include them in project development (e.g. using `curl` or `gh release download` depending 
on repository visibility or authentication requirements).

## Usage 

Like any protobuf file use `protoc` to compile these into any supported language (for example java, c++, TypeScript...).
This may require usage of plugins which can be found [here](https://github.com/protocolbuffers/protobuf/blob/main/docs/third_party.md).

After running the compilation command, the output will contain generated message classes to use in your implementation.
Do not commit those classes as they should always be generated to keep up to date.

Make sure the output directory exists or create it (for example `mkdir -p <out-dir>`) before running `protoc`.

The versioning helps maintaining reproducibility and tracking changes, the `.gitattributes` file sets all other files to be excluded, allowing for minimal overhead when copying into projects. 

## Compilation

### Checking protoc syntax only 

This will output to `/dev/null` therefore not making any result, but will fail if any file has syntax or type error. 

```bash
protoc -I . --descriptor_set_out=/dev/null *.proto
```

### Using TypeScript plugin for protoc 

#### Prerequisites 

1. `npm` or any package manager 
2. Install dependencies (`npm install @bufbuild/protobuf @grpc/grpc-js ts-proto`)
3. existing output directory (read above in [Usage](#usage))
4. source protobuf files (downloaded and unzipped from release section or added via other installation commands) 

```bash
protoc --plugin=protoc-gen-ts_proto=./node_modules/.bin/protoc-gen-ts_proto \
  --ts_proto_out=<out-dir> \
  --ts_proto_opt=outputServices=grpc-js,esModuleInterop=true \
  -I <proto-dir> <proto-dir>/*.proto
```

`*.proto` is a wild-card used to include all the files ending in `.proto` in the compilation.

`<out-dir>` can be something like `src/proto` where `src` is where the rest of the project source code lives (for example in a node.js app). 

`<proto-dir>` can be something like `proto` or `./` in the root of the project or the root project folder (i.e. `.`)

example: 

```bash
protoc --plugin=protoc-gen-ts_proto=./node_modules/.bin/protoc-gen-ts_proto \
  --ts_proto_out=src/proto \
  --ts_proto_opt=outputServices=grpc-js,esModuleInterop=true \
  -I proto proto/*.proto
```