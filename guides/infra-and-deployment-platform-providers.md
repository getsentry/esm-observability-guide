# Cloud Infrastructure Providers and Deployment Platforms

Cloud infrastructure providers and deployment platforms play a crucial role in enabling seamless observability for
modern JavaScript applications. To support ESM-based observability, these platforms must implement flexible runtime and
build-time mechanisms.

## Runtime

Platforms must provide comprehensive support for the `--import` CLI flag, through either:

- Ability to dynamically modify the `node` command to add Node CLI flags
- Ability to specify a path that will be preloaded with `--import` (e.g. through a UI input field)
- Ability to add environment variables scoped to the runtime, which enables adding
  `NODE_OPTIONS=--import './instrumentation.mjs'`

## Build-time

Node.js bundlers (such as nft, zisi) need enhanced functionality to handle modern observability scenarios:

- Automatically detect and include files and packages imported via `module.register`
- Provide flexible configuration for including auxiliary files from the build output like `instrumentation.mjs`
