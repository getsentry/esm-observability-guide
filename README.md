**⚠️ this guide is a work in progress**

# ESM Observability Instrumentation Guide

This guide describes a guide on enabling observability in an application which is using ECMAScript modules.

Observability auto-instrumentation often requires to monkey-patch libraries while they are loaded to automatically
enable instrumentation for them.
As modules are loaded synchronous with `require()`in CommonJS, those modules can be monkey-patched when loaded.

Due to the module resolution algorithm of ESM and ESM not being monkey-patchable, ESM cannot be auto-instrumented like
in CJS.

This guide will outline the current state of ESM on the server-side, explain the problem and propose different
solutions.

## Context

ESM is becoming more and more a default in the server-side build output of frameworks. Here is an overview of popular
Meta-Frameworks and their build output:

| Framework    |                                                                Weekly Download Numbers (rounded) |                                                                      Node Build Output                                                                      |
|--------------|-------------------------------------------------------------------------------------------------:|:-----------------------------------------------------------------------------------------------------------------------------------------------------------:|
| Next.js      |                                                  [8 000 000](https://www.npmjs.com/package/next) | CJS ([source](https://github.com/vercel/next.js/blob/9a1cd356dbafbfcf23d1b9ec05f772f766d05580/packages/next/src/build/webpack-config-rules/resolve.ts#L18)) |
| React Router | [11 000 000](https://www.npmjs.com/package/react-router) <br/>(v7+ framework mode probably 500k) |   ESM ([source](https://github.com/remix-run/react-router/blob/71b4eef6b4f58c8600deb50d5db12af6679235a1/packages/react-router-dev/config/config.ts#L110))   |
| Nuxt         |                                                    [700 000](https://www.npmjs.com/package/nuxt) |         ESM ([source](https://github.com/nuxt/nuxt/blob/f458153d9fda237724c61b1714c56c23221961e1/docs/7.migration/2.configuration.md?plain=1#L101))         |
| SvelteKit    |                                           [450 000](https://www.npmjs.com/package/@sveltejs/kit) |          ESM ([source](https://github.com/sveltejs/kit/blob/7c81ac95c8687b09e2d49bad66528b415fd66bb3/packages/adapter-node/rollup.config.js#L21))           |
| Astro        |                                                   [350 000](https://www.npmjs.com/package/astro) |                  ESM ([source](https://github.com/withastro/astro/blob/0a0b1978a7ea9902174df96852e6a676023cd128/scripts/cmd/build.js#L52))                  |
| Gatsby       |                                                  [250 000](https://www.npmjs.com/package/gatsby) |      CJS ([source](https://github.com/gatsbyjs/gatsby/blob/aa403a4145286782d0989462f9bf3bb1525bc2e3/packages/gatsby/src/utils/webpack.config.js#L828))      |

### Goals and Non-Goals

**Goals**

- Providing a guide for what's needed from a Meta-Framework to make observability instrumentation in ESM work
- Providing a guide for Cloud Infrastructure Providers to provide the necessary infrastructure for ESM-based
  applications

**Non-Goals**

- Providing a step-by-step implementation guide

### Terminology

- **Node Library**: A library designed for use in Node.js environments providing functionality for server-side JS
  applications (e.g. express, mongo, graphql, postgres, ioredis)
- **Observability Library**: A software instrumentation tool that enables developers to track and visualize the flow of
  requests by generating performance traces (e.g. OpenTelemetry)
- **Meta-Framework**: A framework which extends a base JS framework with full-stack capabilities like server-side
  rendering, routing, API endpoints, and an optimized build process (e.g. Next.js, SvelteKit, Nuxt)
- **Cloud Infrastructure Provider**: Large-Scale cloud computing providers offering infrastructure services including
  storage, networking, and advanced cloud technologies (e.g. AWS, Google Cloud, Microsoft Azure)
- **Deployment Platform**: Specialized hosting services offering continuous deployment, content delivery and scaling for
  web applications (e.g. Vercel, Netlify)
- **Web Application Developer**: A software developer who builds web application using different libraries to create a
  JavaScript application

## Problem Statement

As already mentioned in the introduction above, ESM is not monkey-patchable.

A short recap of the ESM module resolution algorithm: ESM loads modules asynchronous as the module resolution algorithm
is split into three phases:

1. Construction (Loading and parsing files into a module record)
2. Instantiation (linking files, creating bindings which is basically "allocating memory space")
3. Evaluation (executing code and storing actual variable values)

The import bindings are created during the 2. phase. After this, the module graph of all statically `import`ed files is
built.
As the module graph is built before doing any evaluation, the static `import`s cannot be monkey-patched on-the-fly (like
it can be done with `require()` in CJS).

However, module loading can be customized
using [customization hooks](https://nodejs.org/docs/v22.13.0/api/module.html#customization-hooks).

## General Implementation Requirements

- The setup code needed for any instrumentation should run **before the rest of the application** (this is needed to
  register
  the [Node Customization Hooks](https://nodejs.org/docs/v22.13.0/api/module.html#customization-hooks),
  observability instrumentation usually relies on for auto-instrumentation to "monkey-patch"). This preloading can be
  achieved by:
    - Passing an instrumentation file to the Node CLI flag `--import`
    - The instrumentation is called at the entry point of the application and everything else is loaded with a dynamic
      `import()`

**OR**

- The Node Libraries are emitting events to the Node Diagnostics channel and the instrumentation picks this up
    - The user is using the minimum version of the Node library which already makes use of the Node Diagnostics Channel

Since the second approach is a great approach to work towards to, it requires implementation of it in each Node library
and then also requires all the people using this library to upgrade to this new version. This means, it's not something
that can possibly be relied on in the upcoming time, but is more of a long-term goal.

## Guidelines

In the following section, guidelines for different stakeholders are presented.

### Meta-Framework Authors

Those are the recommended ways of preloading observability instrumentation (in no particluar order):

1. Defining an optional instrumentation file which users can use to add code which is preloaded.
2. Defining an instrumentation hook which preloads instrumentation code at startup
3. Combination of methods 1 and 2 described above: Defining an instrumentation hook inside an optional instrumentation
   file.

#### 1. Optional instrumentation file

Users add a file called `instrumentation.(ts|mjs)` (optional possibility for different naming through a config option).

The framework will look for this file and will preload its content before the rest of the application runs.

#### 2. Provide an instrumentation hook

Application developers are able to provide their wanted instrumentation inside a certain hook, which preloads code at
application startup.

- This hook can be added in an external file `instrumentaion.ts|mjs`

### Cloud Infrastructure Providers and Deployment Platforms

- **Node Bundlers:** The used Node bundler (nft, zisi, ...) must include files imported with `module.register` in the
  build output.
- **External File:** The used Node Bundler (nft, zisi, ...) must offer the possibility to include "external" files, that
  cannot be detected through tracing the file dependencies (like an external `instrument.ts` which is only passed to
  `--import`)
- **Node CLI flag:** Offer the possibility to add an `--import` CLI flag to the Node command (scoped to the runtime)

### Observability Library Authors

If a Meta-Framework does not include the possibility to preload code, there are two options to set up an observability
library to preload the instrumentation:

- General: Make the web application developers create an external file which sets up the observability library and
  include this file in the build output
- Approach 1: Let the web application developers include the file path from the build output to the Node CLI flag
  `--import`
- Approach 2: Modify the build output to load the observability instrumentation in the entry file and load the rest of
  the application with dynamic `import()`

### Node Library Authors

-> Using Node Diagnostics Channel

This would also enable observability instrumentation in bundled code output.

## Reference

### Further Reading

- [OTel: ECMAScript Modules vs CommonJS](https://github.com/open-telemetry/opentelemetry-js/blob/main/doc/esm-support.md)

### Example Implementations

- **Next.js Instrumentation Hook**
    - [Docs](https://nextjs.org/docs/app/building-your-application/optimizing/instrumentation)
    - [PR which adds this hook](https://github.com/vercel/next.js/pull/46002)
- **OTel**
    - [PR which adds ESM support](https://github.com/open-telemetry/opentelemetry-js/pull/3698)
