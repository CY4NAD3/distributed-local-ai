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
- ✅ `10.0.0.1` static IP made persistent via a NetworkManager profile (manual IPv4). Cause of the earlier drops: the profile was on DHCP with no DHCP server on the point-to-point link
- ⚠️ Still to re-verify: the static IP survives a reboot, and iperf3 still shows ~942 Mbps after the change
- ✅ **Desktop:** llama.cpp built with CUDA + RPC and verified (details below)
- ✅ **Desktop:** single-GPU smoke test passed
- 🔄 **Laptop:** toolchain in progress — driver checked, Git and Visual Studio 2022 Build Tools (MSVC v143) installed; CUDA Toolkit 13.4 and CMake still to install, then clone, checkout and build
- ⏭️ Not started yet: pooled RPC testing, benchmarking, Odysseus setup

See [BUILD_LOG.md](./BUILD_LOG.md) for the full debugging story, commands used, and lessons learned along the way.

## Build reference

Both machines must build the **same llama.cpp commit**, because the RPC protocol changes between versions.

| | Desktop (CachyOS) | Laptop (Windows) |
|---|---|---|
| Pinned commit | `957538960` (build 11141) | `957538960` |
| CUDA Toolkit | 13.4 | 13.4 (planned) |
| Host compiler | GCC 16.2.1 | MSVC v143 (Visual Studio 2022 Build Tools) |
| GPU architecture flag | `75` (Turing) | `120` (Blackwell), planned |

Desktop build:

```bash
git clone https://github.com/ggml-org/llama.cpp && cd llama.cpp
cmake -B build -DGGML_CUDA=ON -DGGML_RPC=ON -DCMAKE_CUDA_ARCHITECTURES=75
cmake --build build --config Release -j6
```

Laptop build (planned, not yet run): same clone, then `git checkout 957538960`, and configure with `-G "Visual Studio 17 2022"` and `-DCMAKE_CUDA_ARCHITECTURES=120`. The VS 2022 toolset is used deliberately, since MSVC v145 from Visual Studio 2026 is still experimental with `nvcc`.

Notes:

- In this llama.cpp version the RPC worker binary is `ggml-rpc-server` (not `rpc-server`). It binds to `127.0.0.1:50052` by default, so it must be pointed at the direct-link IP to be reachable. RPC has no authentication or encryption, so bind it to the private link address only.
- Model files live on an NTFS HDD (`/mnt/1TB`, mounted with `ntfs3`). The disk only affects load time, so the model under test is copied to NVMe.

## Measurements so far

| Test | Setup | Result |
|---|---|---|
| Network | Direct cable, iperf3 | 942 Mbps, 0 retransmits |
| Single-GPU smoke test | RTX 2060 Super, Gemma 3 1B Q4_K_M, `-ngl 99` | 160.2 t/s generation, 252.5 t/s prompt |

The 1B model and short prompt make this a pipeline check, not a real benchmark. A 7–8B single-GPU baseline is still to do. Note that pooling is slower than a single GPU for any model that fits on one card, so pooled tests need a model larger than one card's free VRAM (roughly 6.5 GB usable on the desktop while the desktop session is running).

## Next steps

1. Finish the laptop toolchain: CUDA Toolkit 13.4, CMake, then clone to `D:\odysseus`, check out `957538960`, configure and build.
2. Re-verify the static IP after a reboot and re-run iperf3.
3. Source a second RTL8153-based USB-Ethernet adapter for the laptop, to take the direct RPC cable off the onboard port and protect its hinge-mounted Ethernet jack.
4. Single-GPU baseline with a 7–8B Q4 model on the desktop, then on the laptop.
5. Pooled RPC testing with a model larger than one GPU, then benchmarking.
6. Set up Odysseus and confirm local multi-device access (both PCs + phone).
7. Add a `setup.sh` once the full pipeline works end to end.
