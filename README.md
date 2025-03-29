# Natslive

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

A lightweight, cloud-agnostic event routing layer built on [NATS](https://nats.io).

## Overview

Natslive is a dynamic event router that connects different message sources and destinations with powerful filtering capabilities. Think of it as a self-hosted, lightweight alternative to AWS EventBridge or a simpler version of Knative Eventing, but built natively for NATS.

With WREL (WASM Rule Expression Language), Natslive evolves from a static router into a runtime-extensible event processing engine capable of safely running user-defined filters in WebAssembly sandboxes.

## Key Features

- **Dynamic Routing**: Update rules at runtime via NATS messages
- **Advanced Filtering**: From simple pattern matching to powerful expression languages (NREL & WREL)
- **Multiple Transports**: Connect NATS with HTTP, Kafka, MQTT, and cloud services
- **Lightweight Deployment**: Purpose-built binaries for different environments
- **Cloud-Agnostic**: Works the same across all environments - local, cloud, or hybrid
- **WASM-powered Rule Execution**: Safely run custom filtering logic in isolated sandboxes

## Status

⚠️ **Pre-alpha**: This project is in early design phase. Code implementation is not ready yet.

## Documentation

- [Introduction & Core Concepts](https://github.com/ha1tch/natslive/blob/main/doc/natslive-01-intro.md)
- [Transport Layer Details](https://github.com/ha1tch/natslive/blob/main/doc/natslive-02-transport.md)
- [Rule Expression Language](https://github.com/ha1tch/natslive/blob/main/doc/natslive-03-nrel.md)
- [Binary Bundling Strategy](https://github.com/ha1tch/natslive/blob/main/doc/natslive-04-bundling.md)
- [WASM Rule Expression Language](https://github.com/ha1tch/natslive/blob/main/doc/natslive-05-wrel.md)
- [Comparison with Alternatives](https://github.com/ha1tch/natslive/blob/main/doc/natslive-06-alternatives.md)

## License

Apache License 2.0 - See [LICENSE](LICENSE) for details.