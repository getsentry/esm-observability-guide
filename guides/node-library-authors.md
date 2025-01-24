# Node Library Authors

## 1. Node Diagnostics Channel

The Node Diagnostics Channel provides a high-performance event channel for emitting data, introduced
in [Node.js 15.1.0](https://nodejs.org/en/blog/release/v15.1.0).

Emitting observability data through the diagnostics channel offers several key benefits:

- **Performance**: Lightweight and built directly into Node.js
- **Low Overhead**: Minimal performance impact compared to other instrumentation methods
- **Universal Compatibility**: Works across different module systems (ESM and CJS)

In order to let observability library authors consume those channels, follow the following implementation guidelines:

- Create named channels that represent specific events or operations
- Provide comprehensive documentation detailing:
    - Channel names
    - Event structures
    - Example usage
- Ensure consistent and meaningful data emission

Emitting data through the Node diagnostics channel would also enable observability instrumentation in bundled code
output and there is no need to preload instrumentation with `--import` or dynamic `import()`.

While the Node Diagnostics Channel represents an ideal long-term solution, its widespread adoption faces challenges as
already mentioned in the [general implementation requirements](#general-implementation-requirements) above:

- Requires implementation in the Node.js library
- Requires observability libraries to consume events publish on those channels
- Requires all web application developers using this Node.js library to upgrade to this new version
- Requires all web application developers to upgrade the observability library which is consuming those events

## 2. (optional) Emit OpenTelemetry Spans

Although using the Node diagnostics channel is the preferred way as it's faster, Node library authors can also directly
emit OpenTelemetry Spans themselves. Those spans can be picked up by other observability libraries.

