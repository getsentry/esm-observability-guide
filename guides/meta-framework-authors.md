# Meta-Framework Authors

The core principle is to ensure that observability instrumentation is loaded before rest of the application runs.
This is important as the observability library will register Node.js customization hooks.

The Node customization hooks should be called in the entry point, and dynamic `import()` is used for loading code that
should be run after the hooks are registered. This now dynamically `import()`ed code would be the previous server entry
point.

1. **Additional instrumentation file:** Let web application developers add an optional file `instrumentation.(ts|mjs)` (
   optional possibility for different naming through a config option).
2. **Add file to build output:** The instrumentation file must be present in the server-side build output
3. **Update Server Entry (for preloading instrumentation):** Statically `import` the instrumentation file and
   dynamically `import()` the previous server entry file

## Example

The previous server entry (without instrumentation):

```js 
// build/server/index.mjs
/* Any server code... */
```

Server entry with instrumentation:

```js
// build/server/index.mjs
import './instrumentation.mjs'

import('./server.mjs')
```

Several build tools will inline the content of the instrumentation file, so there will only be the dynamic `import()`
present.

