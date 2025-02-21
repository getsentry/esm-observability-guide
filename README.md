# ESM Observability Instrumentation Guide

This guide describes a guide on enabling observability in an application which is using ECMAScript modules.

[⬇️ Jump straight to Guidelines](#guidelines)

In traditional CommonJS (CJS) module systems, developers could easily "monkey-patch" libraries during loading, allowing
automatic instrumentation for performance tracking, error monitoring, and detailed tracing. The synchronous loading
mechanism of `require()` made this straightforward.

ECMAScript Modules introduce a fundamentally different module resolution algorithm that breaks this traditional
instrumentation approach. Unlike CJS, ESM:

- Loads modules asynchronously
- Builds a static module graph before evaluation
- Prevents on-the-fly modifications to imported modules

This guide will outline the current state of server-side ESM, explain the technical challenges of module instrumentation
and propose solutions for observability in modern Node.js applications.

By the end of this document, developers, framework authors, and infrastructure providers will understand the nuanced
landscape of ESM observability and have concrete strategies for implementation.

## Context

ESM is becoming more and more a default in the server-side build output of frameworks. Here is an overview of popular
Meta-Frameworks and their default build output:

| Framework    |                                                                Weekly Download Numbers (rounded) |                                                                      Node Build Output                                                                      |
|--------------|-------------------------------------------------------------------------------------------------:|:-----------------------------------------------------------------------------------------------------------------------------------------------------------:|
| Next.js      |                                                  [8 000 000](https://www.npmjs.com/package/next) | CJS ([source](https://github.com/vercel/next.js/blob/9a1cd356dbafbfcf23d1b9ec05f772f766d05580/packages/next/src/build/webpack-config-rules/resolve.ts#L18)) |
| React Router | [11 000 000](https://www.npmjs.com/package/react-router) <br/>(v7+ framework mode probably 500k) |   ESM ([source](https://github.com/remix-run/react-router/blob/71b4eef6b4f58c8600deb50d5db12af6679235a1/packages/react-router-dev/config/config.ts#L110))   |
| Nuxt         |                                                    [700 000](https://www.npmjs.com/package/nuxt) |         ESM ([source](https://github.com/nuxt/nuxt/blob/f458153d9fda237724c61b1714c56c23221961e1/docs/7.migration/2.configuration.md?plain=1#L101))         |
| SvelteKit    |                                           [450 000](https://www.npmjs.com/package/@sveltejs/kit) |          ESM ([source](https://github.com/sveltejs/kit/blob/7c81ac95c8687b09e2d49bad66528b415fd66bb3/packages/adapter-node/rollup.config.js#L21))           |
| Astro        |                                                   [350 000](https://www.npmjs.com/package/astro) |                  ESM ([source](https://github.com/withastro/astro/blob/0a0b1978a7ea9902174df96852e6a676023cd128/scripts/cmd/build.js#L52))                  |

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

A short recap of the ESM module resolution algorithm: The ESM module resolution algorithm loads modules asynchronously
as it is split into three phases:

1. Construction (Loading and parsing files into a module record)
2. Instantiation (linking files, creating bindings which is basically "allocating memory space")
3. Evaluation (executing code and storing actual variable values)

The import bindings are created during the 2. phase. After this, the module graph of all statically `import`ed files is
built.
As the module graph is built before doing any evaluation, the static `import`s cannot be monkey-patched on-the-fly (like
it can be done with `require()` in CJS).

However, module loading can be customized
with [customization hooks](https://nodejs.org/docs/v22.13.0/api/module.html#customization-hooks) using `register()` and
`--import`. In observability instrumentation libraries, loader customization and interception is often done
with [import-in-the-middle](https://www.npmjs.com/package/import-in-the-middle) (e.g. in OpenTelemetry).
`import-in-the-middle` works by wrapping modules in dummy modules that enable runtime modifications of exports,
effectively working around ESM's external immutability constraints.

## General Implementation Requirements

- The setup code needed for any instrumentation should run **before the rest of the application** (this is needed to
  register
  the [Node Customization Hooks](https://nodejs.org/docs/v22.13.0/api/module.html#customization-hooks),
  observability instrumentation usually relies on for auto-instrumentation to "monkey-patch"). This preloading can be
  achieved by **either**:
    - Passing an instrumentation file to the Node CLI flag `--import`
    - The instrumentation is called and initialized at the entry point of the application, and everything else (i.e. the
      server code) is loaded with a dynamic `import()`

**OR**

- The Node Libraries are emitting events to the Node Diagnostics channel and the instrumentation picks this up
    - The user must use the minimum version of the Node library which already makes use of the Node Diagnostics Channel

Since the second approach is a great approach to work towards to, it requires a complex, sequential upgrade path:
Node.js library authors must first implement diagnostics channel support, then observability library authors must update
their tools to consume these new events, and finally, every web application developer must upgrade both their Node.js
libraries and observability libraries simultaneously.

## Guidelines

In the following sections, guidelines for different stakeholders are presented.

- [Meta-Framework Authors](./guides/meta-framework-authors.md)
- [Node Library Authors](./guides/node-library-authors.md)
- [Cloud Infrastructure Providers and Deployment Platforms](./guides/infra-and-deployment-platform-providers.md)

## Reference

### Further Reading

- [OTel: ECMAScript Modules vs CommonJS](https://github.com/open-telemetry/opentelemetry-js/blob/main/doc/esm-support.md)

### Example Implementations

- **Next.js Instrumentation Hook**
    - [Docs](https://nextjs.org/docs/app/building-your-application/optimizing/instrumentation)
    - [PR which adds this hook](https://github.com/vercel/next.js/pull/46002)
- **OTel**
    - [PR which adds ESM support](https://github.com/open-telemetry/opentelemetry-js/pull/3698)
