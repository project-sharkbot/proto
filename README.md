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
The exception here is `services.proto` which end at version. The result is like `owner.project.version.Service/Method`.

Versioning is meant to be as following `vM.m.p` (Major, minor, patch) -> `vM` (Major). 
Therefore, any changes that break the current flow should be visible from the major version number. 
Currently, there is no plan to implement a `changelog.md` or maintain git tags that contain this versioning convention 
in this repo. There is an idea to make a pipeline that will create a release with

## Usage 

Like any protobuf file use `protoc` to compile these into any supported language (for example java, c++, TypeScript...).
This may require usage of plugins which can be found [here](https://github.com/protocolbuffers/protobuf/blob/main/docs/third_party.md).

After running the compilation command, the output will contain generated message classes to use in your implementation.
Do not commit those classes as they should always be generated to keep up to date.

The output directory must exist before the command is executed - it is recommended to add a `.gitkeep` in the output 
directory, force add it to a commit, then add the directory to `.gitignore`.

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
2. installed `ts-proto` package along with `proto` package (`npm i proto ts-proto`)
3. existing output directory (read above in [Usage](#usage))
4. source protobuf files 

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