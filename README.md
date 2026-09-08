# Distributed Local AI

A local distributed LLM inference setup using two consumer GPUs across two PCs.

## Goal

Build a local AI system that distributes LLM inference across:

- **Desktop:** RTX 2060 Super 8 GB — CachyOS
- **Laptop:** RTX 5060 8 GB — Windows

The two machines will communicate over a direct Ethernet connection.

The main software stack will be:

- [llama.cpp](https://github.com/ggml-org/llama.cpp) — LLM inference and GPU distribution
- llama.cpp RPC — communication between the two machines
- [Odysseus](https://github.com/apexEvan/odysseus) — user-facing AI interface

## Architecture

```text
                    ┌──────────────────┐
                    │   Phone / PCs    │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │    Odysseus      │
                    │    CachyOS       │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │   llama-server   │
                    │    CachyOS       │
                    └───────┬──────────┘
                            │
                    Direct Ethernet
                            │
                 ┌──────────┴──────────┐
                 ▼                     ▼
        ┌────────────────┐    ┌────────────────┐
        │ RTX 2060 Super │    │    RTX 5060    │
        │    8 GB        │    │     8 GB       │
        │    CachyOS     │    │    Windows     │
        └────────────────┘    └────────────────┘  
