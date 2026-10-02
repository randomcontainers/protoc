# protoc

Container images with protoc, the [Protocol Buffers](https://protobuf.dev/) compiler. protoc reads `.proto` files and generates code for C++, C#, Java, Kotlin, Objective-C, PHP, Python and Ruby, and runs plugins for other languages and for gRPC. It also converts messages between the binary and the text format and writes descriptor sets. protoc is compiled from the protobuf release tarball on Ubuntu and Alpine, for `linux/amd64` and `linux/arm64`, and the images are rebuilt when protobuf publishes a release and when the base image changes.

This is an unofficial build of the open source protobuf, not affiliated with or endorsed by Google or the Protocol Buffers project. Report problems with the image in this repository and protoc bugs [upstream](https://github.com/protocolbuffers/protobuf/issues).

## Quick start

Generate Python modules and type stubs from the `.proto` files in `proto/`:

```sh
mkdir -p gen
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/protoc \
  -I proto --python_out=gen --pyi_out=gen proto/*.proto
```

The shell expands `proto/*.proto` on the host. The paths also work in the container, because the current directory is mounted at `/work`, the container's working directory. protoc does not create output directories, so `gen` has to exist. Imports of the well-known types, such as `google/protobuf/timestamp.proto`, need no `-I`: the image has them in `/usr/local/include`.

The other built-in generators are `--cpp_out`, `--csharp_out`, `--java_out`, `--kotlin_out`, `--objc_out`, `--php_out`, `--ruby_out` and `--rbs_out` (RBS type signatures for Ruby). `--rust_out` is experimental and needs `--rust_opt=experimental-codegen=enabled`. `protoc --help` lists every option.

## Matching the runtime version

Generated code needs a protobuf runtime library at least as new as the protoc that generated it, and C++ code needs the same release. Code from a newer protoc can be rejected by your runtime, so pin the image to the release of your runtime with a `<version>` tag such as `36.2`, rather than `latest` or `36`. The runtimes put their own major version in front of the release: protobuf 36.2 is version 7.36.2 of the C++ and Python runtimes, 4.36.2 for Java and 3.36.2 for C#.

## Plugins

For an `--<name>_out` option that is not built in, protoc runs `protoc-gen-<name>` from the `PATH`. `--plugin=protoc-gen-<name>=<path>` runs a plugin from somewhere else, such as the mounted directory. Plugins run inside the container, which has no Go, Node.js or Java, so a statically linked binary is the simplest kind to add. It works on both the Ubuntu and the Alpine images:

```dockerfile
FROM golang:1.27 AS plugins
RUN CGO_ENABLED=0 GOBIN=/out go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.36.12

FROM ghcr.io/randomcontainers/protoc:36.2
COPY --from=plugins /out/protoc-gen-go /usr/local/bin/
```

In that image, `--go_out=gen` works like a built-in generator. gRPC plugins such as `protoc-gen-go-grpc` are added the same way.

[buf](https://buf.build/) is a separate tool that reads the same `.proto` files with its own compiler and manages plugins and dependencies itself. It is not in this image.

## Descriptor sets and the text format

`--descriptor_set_out` writes the parsed files as a serialized `FileDescriptorSet`, for tools that load a schema at run time. `--include_imports` adds every file they import:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/protoc \
  -I proto --include_imports --descriptor_set_out=api.pb proto/*.proto
```

`--decode=<message type>` reads a binary message on standard input and prints it in the text format, and `--encode` does the reverse. Pass `-i` to `docker run` so the message reaches protoc:

```sh
docker run --rm -i --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/protoc \
  -I proto --decode=demo.Event proto/event.proto < event.bin
```

`--decode_raw` needs no `.proto` file and prints field numbers instead of names.

## What is in the image

- `protoc` in `/usr/local/bin`, linked against the static protobuf libraries it was built with and the distro's shared Abseil libraries.
- In `/usr/local/include/google/protobuf/`: `descriptor.proto`, `compiler/plugin.proto`, the well-known types (`any`, `api`, `duration`, `empty`, `field_mask`, `source_context`, `struct`, `timestamp`, `type` and `wrappers`) and the feature definitions used with editions, such as `cpp_features.proto` and `java_features.proto`. protoc finds them in the `include` directory next to its own `bin` directory.

Not included: the protobuf runtime libraries and C++ headers, the upb generators (`protoc-gen-upb` and the others) and plugins for languages without a built-in generator, such as Go, Dart, Swift and JavaScript. The CMake options, the compiler and the Abseil version are in `/usr/local/share/randomcontainers/protoc/buildinfo`.

## Default or slim

protoc's default image adds no other tools, so `latest` and `slim` are the same image, with the contents listed above. Use `latest` to run it and the `slim` tags as a base for your own image.

## Tags

`<version>` is a protobuf release such as `36.2`. `<major>` is its shorter form, `36`, and follows the newest release in that series. Each row lists the default tag and its `slim` twin, which point to the same image.

| Tags | Base |
|---|---|
| `latest`, `slim` | Ubuntu |
| `<version>`, `<version>-slim` | Ubuntu |
| `<major>`, `<major>-slim` | Ubuntu |
| `ubuntu`, `slim-ubuntu` | Ubuntu |
| `<version>-ubuntu`, `<version>-slim-ubuntu` | Ubuntu |
| `<major>-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04`, `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `slim-alpine` | Alpine |
| `<version>-alpine`, `<version>-slim-alpine` | Alpine |
| `<major>-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24`, `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current protobuf version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

protoc writes generated code, descriptor sets and encoded messages into the mounted directory, so run it with the flag above.

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new protobuf release and are rebuilt when the base image changes; `<version>-slim-ubuntu` and `<version>-slim-alpine` stay on one release. Besides plugins in `/usr/local/bin` (see [Plugins](#plugins)), an image can carry `.proto` files that your projects import. protoc searches `/usr/local/include` after the directories given with `-I`:

```dockerfile
FROM ghcr.io/randomcontainers/protoc:36.2-slim-ubuntu@sha256:...
COPY googleapis/google/api/ /usr/local/include/google/api/
```

To install distro packages, switch to `USER root` and back to `USER 1000:1000` afterwards. The entrypoint is `["tini", "--", "protoc"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/protoc/` holds the version, the source URL, the build options, the license files and `runtime-deps`, the list of distro packages protoc needs at run time.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/protoc:latest \
  --repo randomcontainers/protoc --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/protoc:latest --format '{{ json .SBOM }}'
```

protobuf publishes no signature or checksum file for its `protobuf-<version>.tar.gz` source tarball. The SHA-256 in `package.yml` is the digest GitHub records for that release asset. Before compiling, the build checks the tarball against it. CMake configures and compiles without network access, so nothing else is downloaded.

## Updates

The project checks the releases of [protocolbuffers/protobuf](https://github.com/protocolbuffers/protobuf/releases) every 15 minutes and skips release candidates. A release is picked up once it is 24 hours old. Its source tarball is checked against the digest GitHub records for the release asset, the new version and the tarball's SHA-256 are committed to `package.yml`, and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built. Fixes that protobuf releases for an older series, such as 34.2 after 35.0, are not built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  -t protoc:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs.

## Licenses

protobuf is licensed under the three-clause BSD license (BSD-3-Clause). protoc also contains utf8_range, a UTF-8 validator under the MIT license, and code compiled in from Abseil's headers (inline functions and templates) under the Apache License 2.0, so the image's license label is `BSD-3-Clause AND Apache-2.0 AND MIT`. The license files are in `/usr/local/share/randomcontainers/protoc/licenses/`: protobuf's `LICENSE`, utf8_range's `LICENSE.utf8_range`, and `NOTICES`, the third-party notices that `protoc --notices` prints, which include the Apache License text. The source tarball URL is in `/usr/local/share/randomcontainers/protoc/source`. Abseil's shared libraries, which protoc loads at run time, come from the Ubuntu or Alpine packages and keep their own license, as do the other distro packages in the image. The SBOM lists them.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
