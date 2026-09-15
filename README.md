# Distributed Local AI (Project Odysseus)

A local distributed LLM inference setup pooling GPU VRAM across two consumer machines, so models larger than a single 8GB GPU can run entirely on local hardware.

## Goal

Build a local AI system that distributes LLM inference across:

- **Desktop (main node):** Ryzen 5 2600X, RTX 2060 Super 8GB, 16GB RAM — CachyOS
- **Laptop (RPC worker):** Ryzen 7, RTX 5060 8GB, 16GB RAM — Windows

The two machines communicate over a direct Ethernet link, pooling **16GB of combined VRAM** to run models neither GPU could hold alone.

Software stack:

- [llama.cpp](https://github.com/ggml-org/llama.cpp) — inference engine, built with CUDA + RPC backend
- llama.cpp RPC — communication layer between the two machines
- [Odysseus](https://github.com/apexEvan/odysseus) — user-facing AI interface, run locally and reachable from both machines and phone

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
                    Direct Ethernet (RPC)
                            │
                 ┌──────────┴──────────┐
                 ▼                     ▼
        ┌────────────────┐    ┌────────────────┐
        │ RTX 2060 Super │    │    RTX 5060    │
        │    8 GB        │    │     8 GB       │
        │    CachyOS     │    │    Windows     │
        └────────────────┘    └────────────────┘
```

### Network layout

| Link | Path | Result |
|---|---|---|
| RPC link (desktop ↔ laptop) | CachyOS `enp3s0` (`10.0.0.1/24`) → cat6e direct cable → Legion `10.0.0.2` | 942 Mbps, zero retransmits (iperf3) |
| Desktop internet | Onten OTN-5225D USB-Ethernet adapter (RTL8153) → Cudy M1800 router | Gigabit link, ~109/79 Mbps (matches ISP plan) |
| Laptop internet | WiFi | `192.168.10.151` |

## Hardware summary

| Component | Desktop (main node) | Laptop (RPC worker) |
|---|---|---|
| CPU | Ryzen 5 2600X | Ryzen 7 |
| GPU | RTX 2060 Super 8GB | RTX 5060 8GB |
| RAM | 16GB | 16GB |
| OS | CachyOS (Arch-based) | Windows |
| Role | Odysseus + llama-server host | RPC worker |

Network hardware: Cudy M1800 router (main unit), cat6e direct cable, Onten OTN-5225D USB-Ethernet adapter (RTL8153).

## Current status

- ✅ Network architecture finalized and validated — direct cable RPC link (942 Mbps) + separate internet paths per machine
- ⚠️ Open item: the `10.0.0.1` static IP is currently set with `ip addr add`, which doesn't survive a reboot — needs a persistent NetworkManager profile
- ⏭️ Not started yet: llama.cpp build (CUDA + RPC), GPU pooling tests, Odysseus setup

See [BUILD_LOG.md](./BUILD_LOG.md) for the full debugging story, commands used, and lessons learned along the way.

## Next steps

1. Make the `10.0.0.1` static IP persistent via a NetworkManager connection profile.
2. Source a second RTL8153-based USB-Ethernet adapter for the Legion laptop, to take the direct RPC cable off the onboard port and protect its hinge-mounted Ethernet jack.
3. Build llama.cpp with CUDA + RPC support on both machines.
4. Individual GPU testing → pooled RPC testing → benchmarking.
5. Set up Odysseus and confirm local multi-device access (both PCs + phone).
