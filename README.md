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

- ✅ **Network architecture finalized and reboot-validated** — direct cable RPC link (942 Mbps, 0 retransmits) + separate internet paths per machine. The `10.0.0.1/24` static IP is persistent through reboot via NetworkManager.
- ✅ **Desktop:** llama.cpp built with CUDA + RPC and verified
- ✅ **Laptop:** llama.cpp built with CUDA + RPC and verified (`--list-devices` sees the RTX 5060)
- ✅ **Model storage standardized** — Gemma 3 1B, Llama 3.1 8B, and Qwen2.5 14B models are kept under `/mnt/1TB/odysseus-models` on the desktop; the direct link is also used for transferring models to Legion for solo baselines.
- ✅ **Single-GPU baselines captured on both GPUs** — 8B: 59.98 t/s on the 2060 Super vs. 68.27 t/s on the 5060. The 14B model hard-OOMs on CachyOS and technically loads on Windows only through WDDM system-memory spillover at 3.38 t/s.
- ✅ **RPC pooling works end to end** — both GPUs confirmed genuinely participating (`--list-devices` shows `CUDA0` + `RPC0`, `ldd` confirms the RPC backend is linked, and forcing a low `--tensor-split` ratio OOMs the local GPU exactly as expected, proving it holds a real share of the model).
- ✅ **8B tensor-split sweep completed** — `0.66,1` is the practical floor for the desktop GPU's share; changing the ratio within the working range barely changes generation speed because the default pipelined layer split is limited by the slower GPU.
- ✅ **14B pooled milestone achieved** — Qwen2.5-14B-Instruct-Q4_K_M runs across both GPUs at **33.65 ± 0.19 t/s generation** and **634.99 ± 4.81 t/s prompt processing**. This is the first model in the project that neither GPU can cleanly run alone at usable speed.
- ✅ **`-sm row` diagnosis completed** — RPC0 does not support split buffers in this llama.cpp version, so `-sm layer` (the default) is the usable RPC split mode; `-sm tensor` would hit the same backend limitation.
- ✅ **Odysseus workspace configured on CachyOS** and connected to the working pooled `llama-server` via `LLM_ENDPOINTS=http://localhost:8081/v1`
- ✅ **Odysseus local interface verified** — reachable from the desktop, Legion, and phone over the LAN on port `7000`
- ✅ **End-to-end Odysseus inference validated** with the pooled backend, including the Qwen2.5-14B setup; final working configuration and commands are documented

See [BUILD_LOG.md](./BUILD_LOG.md) for the full debugging story, commands used, benchmark data, and lessons learned along the way.

## Build reference

Both machines must build the **same llama.cpp commit**, because the RPC protocol changes between versions.

| | Desktop (CachyOS) | Laptop (Windows) |
|---|---|---|
| Pinned commit | `957538960` (build 11141) | `957538960` |
| CUDA Toolkit | 13.4 | 13.4 |
| Host compiler | GCC 16.2.1 | MSVC v143 (Visual Studio 2022 Build Tools) |
| GPU architecture flag | `75` (Turing) | `120` (Blackwell) |
| Build status | ✅ Built and verified | ✅ Built and verified |

Desktop build:

```bash
git clone https://github.com/ggml-org/llama.cpp && cd llama.cpp
cmake -B build -DGGML_CUDA=ON -DGGML_RPC=ON -DCMAKE_CUDA_ARCHITECTURES=75
cmake --build build --config Release -j6
```

Laptop build:

```powershell
cd D:\odysseus
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp
git checkout 957538960
cmake -B build -G "Visual Studio 17 2022" -DGGML_CUDA=ON -DGGML_RPC=ON -DCMAKE_CUDA_ARCHITECTURES=120
cmake --build build --config Release -j
```

The VS 2022 toolset is used deliberately, since MSVC v145 from Visual Studio 2026 is still experimental with `nvcc`.

Notes:

- In this llama.cpp version the RPC worker binary is `ggml-rpc-server` (not `rpc-server`). It binds to `127.0.0.1:50052` by default, so it's pointed at the direct-link IP explicitly (`-H 10.0.0.2 -p 50052`) to be reachable. RPC has no authentication or encryption, so it's bound to the private link address only and firewalled to just the desktop's IP.
- Model files live on an NTFS HDD (`/mnt/1TB/odysseus-models`, mounted with `ntfs3`). This only costs load time (~35–50s for a 5GB model); once weights are in VRAM, inference speed is identical to running from NVMe.

## Measurements so far

### Network

| Test | Result |
|---|---|
| Satellite wired path | ~94 Mbps, 0 retransmits — Fast Ethernet negotiation on the satellite LAN port |
| Satellite bridge | ~154 Mbps average, frequent retransmits |
| Direct WiFi | ~224 Mbps, frequent retransmits |
| **Direct cable RPC link** | **942 Mbps, 0 retransmits**, reboot-persistent |
| Direct-link latency | 1.98 / 2.19 / 2.48 ms min/avg/max, 0% loss |
| USB adapter → main router | ~109/79 Mbps ISP-capped; adapter negotiates full gigabit |

### Inference

| Test | Setup | Backend | pp512 | tg128 |
|---|---|---|---:|---:|
| Solo baseline, 8B | CachyOS, 2060 Super | CUDA | 1415.00 ± 13.77 | 59.98 ± 0.03 |
| Solo baseline, 8B | Legion, 5060 | CUDA | 2768.76 ± 58.82 | 68.27 ± 0.27 |
| Solo baseline, 14B | CachyOS, 2060 Super | — | **fails: cudaMalloc OOM** | — |
| Solo baseline, 14B | Legion, 5060 | CUDA | 168.05 ± 1.82 | **3.38 ± 0.02** (WDDM memory spillover) |
| Pooled, 8B, even split | CachyOS client + Legion worker | CUDA,RPC | 1222.76 ± 18.40 | 60.01 ± 0.04 |
| Pooled, 8B, split sweep 0.66–0.70 | CachyOS client + Legion worker | CUDA,RPC | not re-measured | 59.57–59.73 |
| **Pooled, 14B** | CachyOS client + Legion worker | **CUDA,RPC** | **634.99 ± 4.81** | **33.65 ± 0.19** |
| Pooled, 14B, `-sm row` | CachyOS client + Legion worker | — | **fails: RPC0 doesn't support split buffers** | — |

The 1B model was only a smoke test: **160.2 t/s generation, 252.5 t/s prompt processing** on the 2060 Super. The 8B model fits on the desktop alone, so pooled 8B generation does not become faster; it tracks the slower 2060 Super rather than summing both GPUs' throughput. The 14B result is the meaningful pooling milestone: neither GPU can cleanly provide a usable solo run, while the pooled run reaches 33.65 t/s.

## Next steps

1. **Optional hardware refinement:** source the second RTL8153-based USB-Ethernet adapter for Legion. This is **not blocking** the build; it would free the hinge-mounted onboard Ethernet port from repeated cable use.
2. **Automation:** add a `setup.sh` (or equivalent setup documentation) to make the final working pipeline reproducible from a clean setup.
