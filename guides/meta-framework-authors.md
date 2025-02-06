# Meta-Framework Authors

The core principle is to ensure that observability instrumentation is loaded before rest of the application runs.
This is important as the observability library will register Node.js customization hooks.

The observability instrumentation should be called and initialized in the entry point of the server. This will register
the Node customization hook. Then a dynamic `import()` is used to load code to be executed after the hooks are
registered. This essentially means that the code that is now dynamically `import()`ed is the previous server entry
point (see example code below).

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

Server entry file with instrumentation:

```js
// build/server/index.mjs
import './instrumentation.mjs';

import('./server.mjs'); // new file which contains code from previous index.mjs file
```

Several build tools will inline the content of the `instrumentation.mjs` file, so there won't necessarily be an external
`instrumentation.mjs` file after building.

