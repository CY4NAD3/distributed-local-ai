# Build Log — Project Odysseus

The full debugging story behind the network layer and the llama.cpp build — including the dead ends and every command used. See [README.md](./README.md) for the project overview and current status.

## Hardware

**CachyOS Desktop** (main node)
- CPU: Ryzen 5 2600X · GPU: RTX 2060 Super 8GB · RAM: 16GB
- Username `azraf`, hostname `AzureLinux`
- Shell: fish

**Legion 5** (RPC worker)
- CPU: Ryzen 7 · GPU: RTX 5060 Laptop GPU 8GB (reports 8123 MiB) · RAM: 16GB · OS: Windows, username `CY4NAD3`

Note: the two GPUs are different generations (Turing vs. Blackwell-class) — compute capability mismatch is a variable to watch during inference benchmarking, since the cards won't process synchronized layers at the same rate. CUDA arch flags differ accordingly: `75` for the 2060 Super, `120` for the 5060.

## 1. Attempt 1 — Cudy M1800 satellite as a local switch (abandoned)

Original plan: wire both CachyOS (Cat5e) and Legion (Cat6) into the **Cudy M1800 mesh satellite's LAN ports** in the same room, letting the satellite switch between them locally while its wireless backhaul to the main router handled internet for both. Looked fine on paper — 4K streamed fine on both machines through it.

`iperf3` benchmarking told a different story:

| Test | Result | Notes |
|---|---|---|
| CachyOS → Legion, both wired to satellite | ~94 Mbps flat, 0 retries | Looked like a wireless bottleneck, wasn't |
| CachyOS on main router, Legion on satellite | ~154 Mbps avg, 55–368 Mbps range, frequent retransmits | Genuine wireless backhaul instability |
| Legion on direct WiFi (no satellite bridging) | ~224 Mbps | Better, still unstable, still far below gigabit |

The flat ~94 Mbps turned out not to be a wireless issue at all — `ethtool` on the satellite's LAN port showed `Speed: 100Mb/s`, i.e. **Fast Ethernet negotiation**, despite the port supporting 1000baseT.

**False alarm mid-debugging:** after moving CachyOS to the main router, `ping`/`iperf3` to Legion returned 100% packet loss ("Destination host unreachable"), even though ARP resolved the MAC fine (ruling out an L2 problem). Root cause: **Legion's Ethernet adapter was classified as a "Public" network in Windows**, which blocks inbound ICMP/TCP by default.

```powershell
Set-NetConnectionProfile -InterfaceAlias "Ethernet" -NetworkCategory Private
```

This exact issue recurred later on a second interface — see step 2. It was the single most time-consuming red herring in the whole process.

**Conclusion:** the satellite path — wired or wireless — couldn't deliver anywhere near gigabit, and the instability (retransmits, jitter) specifically hurts RPC's per-token network chatter, not just raw throughput.

## 2. Attempt 2 — Direct cable link (adopted)

Ran a single cat5e/cat6e cable **directly between CachyOS's onboard NIC and Legion's onboard NIC**, bypassing the satellite and any switch entirely. No DHCP on a point-to-point link, so static IPs:

- CachyOS `enp3s0`: `10.0.0.1/24`
- Legion Ethernet: `10.0.0.2/24`

The same "Public network profile" firewall issue appeared again on this fresh interface and was fixed the same way. Once resolved:

**Result: 942 Mbps sustained, zero retransmits, for the full 30-second test — reproduced identically on a later re-test after unrelated network reconfiguration.** Essentially gigabit line-rate.

## 3. Final architecture — dual-NIC on CachyOS

The direct cable solved the RPC link but left CachyOS with no internet (its one onboard NIC was now dedicated to Legion). Solved with a second NIC:

- **`enp3s0`** (onboard, static `10.0.0.1`) → direct cat6e cable → **Legion onboard NIC** (`10.0.0.2`) — dedicated, isolated RPC link. Validated at 942 Mbps.
- **USB-to-Ethernet adapter** (Onten OTN-5225D, Realtek **RTL8153** chipset, USB 3.0, shows as `enp1s0f0u1`, confirmed via `lsusb` ID `0bda:8153`) → cat6e cable → **main Cudy M1800 router directly** (satellite no longer used at all). `ethtool` confirms `1000Mb/s` link negotiation; real throughput matches the ISP plan (~109/79 Mbps via `speedtest-cli`/fast.com — ISP-capped, not adapter-capped).
- **Legion's internet**: via WiFi (`192.168.10.151`), independent of the RPC link — its onboard Ethernet port is fully dedicated to the direct cable.

**Chipset choice (RTL8153):** deliberate — native, well-supported in-kernel Linux driver (`r8152`) with no manual driver install, plus native Windows support. Same reasoning was applied when later considering WiFi chipsets for other testing (RTL8153 vs. out-of-tree alternatives like RT5572/RTL8822BU — in-tree support wins for reliability).

**Pricing sanity check:** paid ~1050 tk for the Onten vs. ~250–300 tk for unbranded alternatives seen online. Judged reasonable — a genuine branded RTL8153 chipset with proven 942 Mbps/zero-retransmit performance vs. unknown, no-warranty clone chipsets at the bargain end. A ~379 tk "No Brand" listing was evaluated and passed on specifically for the unverified-chipset risk, since this adapter needs to be a reliable "set and forget" long-term part.

**Explored but not needed:** powerline adapters, wired mesh backhaul, a second gigabit switch — direct cable + dual-NIC solved it more simply than any of these.

**Static IP persistence:** the original `10.0.0.1/24` on `enp3s0` was set live via `sudo ip addr add`, which doesn't survive a NetworkManager restart/reboot. Replaced with a proper `nmcli` connection profile so the address comes back automatically. Reboot re-verification (`ip -br addr show enp3s0` should still read `10.0.0.1/24`, plus an `iperf3` re-check at ~942 Mbps) is **still not confirmed**. On 2026-09-24, `ping -c 3 10.0.0.2` from CachyOS returned 3/3 replies (0% loss, 1.98/2.19/2.48 ms min/avg/max), so the link and both addresses are up, but that isn't the reboot test — do that before calling this fully closed.

## Next up (network)

Buying a **second RTL8153-based USB-Ethernet adapter for Legion**, sourced in person through an ISP technician contact rather than an unbranded online listing. This will take over the direct RPC cable on Legion's end, freeing Legion's onboard Ethernet port (a fragile hinge-mounted connector) from repeated plug/unplug wear. Not required to keep building — the current onboard-to-onboard link is what's in use.

## 4. Inference layer — llama.cpp build (both machines built and verified; pooled run done, layer split confirmed)

With the network link validated, moved on to building `llama.cpp` with CUDA + RPC support on both machines, so CachyOS can run `llama-server` as the main node and Legion can run `ggml-rpc-server` as a worker, pooling the 2060 Super and 5060 into one inference target.

The subsections below are grouped per machine, but the work was interleaved. Actual order of events (2026-09-23 → 2026-09-25, the first session ran past midnight):

1. CachyOS: checked the existing CUDA install (already present from hashcat work).
2. CachyOS: installed `cmake`.
3. CachyOS: PATH mishap from running a bash `export` line in fish; fixed.
4. CachyOS: storage planning (free space, NTFS drive mounts, where models will live).
5. CachyOS: cloned llama.cpp, pinned commit `957538960`, configured (clean).
6. CachyOS: started the compile (`-j6`).
7. **While the compile ran:** Legion toolchain work — driver check, found Visual Studio 2026 (v145 toolset only), installed Visual Studio 2022 Build Tools (v143).
8. CachyOS: compile finished; verified binaries, RPC server and GPU visibility.
9. CachyOS: single-GPU smoke test with a 1B model.
10. CachyOS: found where `-hf` had put the model (Hugging Face cache, not `~/odysseus/models`); copied it out with `cp -L`; decided to keep all models on the HDD.
11. CachyOS: downloaded Llama 3.1 8B Q4_K_M to `/mnt/1TB/odysseus-models/` and ran the single-GPU `llama-bench` baseline (pp512 1415 t/s, tg128 59.98 t/s).
12. Legion: installed CMake (winget), then CUDA Toolkit 13.4 (custom install, driver unticked); verified `nvcc` and the Visual Studio integration files.
13. Legion: cloned llama.cpp to `D:\odysseus`, checked out `957538960`, configured with arch `120`, built; `llama-server --list-devices` sees the 5060.
14. Legion: added the firewall rule for TCP 50052, started `ggml-rpc-server` bound to `10.0.0.2`.
15. CachyOS: `ping` and a raw TCP port check to `10.0.0.2:50052` both succeeded.
16. CachyOS: first pooled `llama-bench` run with `--rpc` (pp512 1223 t/s, tg128 60.01 t/s). At this point how the layers were actually split was not yet verified.
17. CachyOS: confirmed the split is real — `--rpc 10.0.0.2:50052 --list-devices` lists both `CUDA0` (2060 Super) and `RPC0` (Legion), and `ldd ./build/bin/llama-server | grep -i rpc` shows `libggml-rpc.so.0` linked.
18. CachyOS: swept `--tensor-split` on the 8B model to find the OOM boundary on the 2060 Super and check whether the split ratio moves throughput. See section 5.

### CachyOS

Already had the CUDA *toolkit* installed from earlier hashcat work (not just the driver), which skipped a step:

```
cuda 13.4.2-1
nvidia-utils 615.71.09-1
nvcc: release 13.4, V13.4.92
```

`nvidia-smi` also showed the desktop session (KDE, browser, Steam, Telegram) holding **1.3 GB of the 8 GB** VRAM on the 2060 Super at idle. Two views of the same headroom: `nvidia-smi` implies ~6.9 GB free, while llama.cpp itself reports `7798 MiB` total and `6541 MiB` free. Use the llama.cpp figure (~6.5 GB) when picking model sizes, and close heavy GPU apps (browser, Steam) before benchmark runs for consistent numbers.

Build steps run:
```bash
sudo pacman -S --needed base-devel cmake git   # cmake wasn't installed yet (~103 MB, v4.4.3); base-devel/git were no-ops
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp
git rev-parse --short HEAD                     # 957538960 — must match on Legion, RPC protocol is version-sensitive
cmake -B build -DGGML_CUDA=ON -DGGML_RPC=ON -DCMAKE_CUDA_ARCHITECTURES=75
cmake --build build --config Release -j6
```
- `GGML_CUDA=ON` compiles the GPU kernels (without it, CPU-only build).
- `GGML_RPC=ON` compiles the RPC server, letting `llama-server` treat a remote GPU as another device.
- `CMAKE_CUDA_ARCHITECTURES=75` targets the 2060 Super (Turing); Legion's 5060 needs `120`.
- `-j6` instead of `-j$(nproc)` — CUDA compilation is memory-hungry, and 16 GB RAM can swap under full parallelism.

Configure output was clean: CUDA Toolkit 13.4.92 found, CUDA and RPC backends included, `Configuring done`. Two harmless warnings: `ccache` not found (only slows rebuilds) and NCCL not found (that concerns multiple GPUs inside one machine, not RPC across two).

**Shell gotcha:** `export PATH=/opt/cuda/bin:$PATH` is bash syntax. Run in **fish** (CachyOS's default shell), it broke PATH for that terminal tab — `uname` and `id` became "unknown command" and the prompt plugins spammed errors. Nothing persistent was affected and a fresh tab worked normally. The fish equivalent is `fish_add_path /opt/cuda/bin`, which reported the directory was already on PATH, so the line was never needed on this machine.

**Build result:** compile finished and verified.
- `./build/bin/llama-server --version` → `0.4.1-dev (build 11141, commit 957538960)`.
- The RPC worker binary is named **`ggml-rpc-server`** in this version, not `rpc-server` (the old name had been assumed from memory; `ls build/bin` showed the real one). Its defaults: `--host 127.0.0.1`, `--port 50052`, `-c` to enable a local file cache, `-t` CPU threads, `-d` device list.
- `libggml-cuda.so` was built (67 MB, sm_75 only) and `./build/bin/llama-server --list-devices` reports `CUDA0: NVIDIA GeForce RTX 2060 SUPER (7798 MiB, 6541 MiB free)`.

**Single-GPU smoke test:**
```bash
./build/bin/llama-cli -hf ggml-org/gemma-3-1b-it-GGUF:Q4_K_M -ngl 99 -p "Explain what a GPU does in two sentences."
```
- `-hf <user>/<repo>:<quant>` downloads the model from Hugging Face and runs it. `-ngl 99` puts all layers on the GPU.
- Result: **160.2 t/s generation, 252.5 t/s prompt processing.** Caveat: a 1B model with a ~12-token prompt is a pipeline check, not a benchmark (prompt speed in particular is noisy at that length).
- First attempt failed because the placeholder `<file>` from an example command was pasted literally; the shell treats `<` as input redirection.

**Where `-hf` puts models (found the hard way):** the download did not land in `~/odysseus/models`. This build uses the Hugging Face cache layout: `~/.cache/huggingface/hub/models--ggml-org--gemma-3-1b-it-GGUF/`. The file under `snapshots/<hash>/` is a **symlink** to the real file in `blobs/`, so a plain `mv` would move only the link. `cp -L` follows the link and copies the real file (806M for the 1B model):
```bash
find ~/.cache -iname "*gemma-3-1b*"
cp -L ~/.cache/huggingface/hub/models--ggml-org--gemma-3-1b-it-GGUF/snapshots/*/gemma-3-1b-it-Q4_K_M.gguf ~/odysseus/models/
./build/bin/llama-cli -m ~/odysseus/models/gemma-3-1b-it-Q4_K_M.gguf -ngl 99 -p "..."   # -m = load a file directly, no download logic
```
From here on, models are downloaded manually (`wget -c`) into a folder chosen on purpose and loaded with `-m`. This also gives the same path on both machines.

**Storage:** `/home` has ~71 GB free (btrfs). The NTFS drives are already auto-mounted through `/etc/fstab` with the in-kernel `ntfs3` driver (`rw,nofail,uid=1000,gid=1000`): `/mnt/1TB` (HDD, drive label "1 TB Drive", ~213 GB free) and `/mnt/Records`. `mount | grep 1TB` confirmed `/dev/sdb1 on /mnt/1TB type ntfs3 (rw,...,uid=1000,...)`, so no permission problems. **Plan changed:** all models now live on the HDD in `/mnt/1TB/odysseus-models/` instead of being copied to NVMe for benchmarks — the HDD only affects model *load* time (roughly 35–50 s for the 4.9 GB 8B file, more for bigger models), not inference speed, and NVMe space is limited. The NVMe "SSD samsung" partition (`nvme0n1p5`, ~99 GB free) is mounted by the desktop under `/run/media/`, a path with a space that isn't stable, so it was left out. The label ("1 TB Drive", shown in Dolphin) and the mount point (`/mnt/1TB`, used in the terminal) are two names for the same drive. The 1B smoke-test file is still in `~/odysseus/models`; moving it to the HDD is planned.

**8B download and single-GPU baseline:**
```bash
mkdir -p /mnt/1TB/odysseus-models
cd /mnt/1TB/odysseus-models
wget -c https://huggingface.co/bartowski/Meta-Llama-3.1-8B-Instruct-GGUF/resolve/main/Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf
cd ~/odysseus/llama.cpp
./build/bin/llama-bench -m /mnt/1TB/odysseus-models/Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf -ngl 99
```
- Download: 4,920,739,232 bytes (4.58 GiB) in 6m 2s at ~13 MB/s — the ISP line was the limit, not the HDD. `-c` resumes an interrupted download.
- Result: **pp512 1415.00 ± 13.77 t/s, tg128 59.98 ± 0.03 t/s** on the RTX 2060 Super. Generation is memory-bandwidth-bound: at ~448 GB/s and ~4.9 GB of weights read per token, the theoretical ceiling is roughly 90 t/s, so ~60 t/s is a healthy result. This is the single-GPU baseline for all pooled comparisons.

### Legion (Windows)

Toolchain problem surfaced immediately: Legion had **Visual Studio 2026** installed, but it only ships the newer **v145 MSVC toolset** (resolved version `14.51.36231`), which is experimental with `nvcc`. No `v143` toolset — the one NVIDIA validates against — was present. (A forum report also described `nvcc` from CUDA 13.4.1 rejecting the VS 2026 toolchain outright.)

Detection detail: `vswhere -latest` first returned only SQL Server Management Studio, which shares the Visual Studio installer but has no C++ compiler. Filtering with `-requires Microsoft.VisualStudio.Component.VC.Tools.x86.x64` is what found the VS 2026 Community install.

Separately, Legion also has **Code::Blocks** installed (used previously for GLUT/Computer Graphics coursework), which bundles **MinGW** (a `g++` port). This doesn't help here: on Windows, `nvcc` only works with MSVC, not MinGW. It also creates a silent risk — if MinGW's `bin` is on `PATH`, CMake could pick it up instead of MSVC by accident. Mitigation: name the compiler explicitly at configure time with `-G "Visual Studio 17 2022"` so CMake can't guess wrong. Code::Blocks itself is untouched by any of this.

Fix: installed **Visual Studio 2022 Build Tools** (the v143 line) side by side with VS 2026, without touching the existing install:
```powershell
winget install Microsoft.VisualStudio.2022.BuildTools --override "--passive --wait --add Microsoft.VisualStudio.Workload.VCTools --includeRecommended"
```
Confirmed installed under a separate path (`Program Files (x86)`, distinct from the VS 2026 location) at toolset `14.44.35207`:
```
C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Tools\MSVC\14.44.35207
```

Driver: Legion is already on **616.92** (reports CUDA 13.4 support), which covers CUDA 13.4 — no driver change needed. Git 2.54.0 was already installed.

**CMake install:** `winget install Kitware.CMake` installed v4.4.3 successfully, but `cmake` was "not recognized" in the PowerShell window that was already open — a window only reads PATH when it starts. A fresh window fixed it (`cmake version 4.4.3`).

**CUDA Toolkit 13.4 install (done):** downloaded the Windows x86_64 local `.exe` installer from NVIDIA and ran it as **Custom**, with the Display Driver components **unticked** (616.92 already covers 13.4) and **Visual Studio Integration ticked**, at the default path on `C:`. Verification:
- Before install, `Test-Path "C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA"` returned `False` and `nvcc` wasn't found — expected, not a PATH problem.
- The installer summary said "Not Installed: Nsight for Visual Studio 2022 — VS2022 was not found". Harmless: Nsight is a debugging add-on for the full IDE, and the 2022 install is Build Tools only. llama.cpp doesn't need it.
- The part that matters is the Visual Studio integration. `Get-ChildItem` on `...\2022\BuildTools\MSBuild\Microsoft\VC\v170\BuildCustomizations -Filter "CUDA*"` listed `CUDA 13.4.props`, `CUDA 13.4.targets`, `CUDA 13.4.Version.props` and `CUDA 13.4.xml`, so the integration attached to the v143 Build Tools.
- In a new PowerShell window, `nvcc --version` → `release 13.4, V13.4.92`.

**Legion clone, configure and build (done):**
```powershell
cd D:\
mkdir odysseus; cd odysseus
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp
git checkout 957538960
git log -1 --oneline        # 957538960 (HEAD) — matches CachyOS
cmake -B build -G "Visual Studio 17 2022" -DGGML_CUDA=ON -DGGML_RPC=ON -DCMAKE_CUDA_ARCHITECTURES=120
cmake --build build --config Release -j
.\build\bin\Release\llama-server.exe --list-devices
```
- Configure found the CUDA Toolkit at `C:/Program Files/NVIDIA GPU Computing Toolkit/CUDA/v13.4/include` (version 13.4.92) and wrote the build files to `D:/odysseus/llama.cpp/build`.
- `--list-devices` → `CUDA0: NVIDIA GeForce RTX 5060 Laptop GPU (8123 MiB, 7043 MiB free)`. Windows itself takes roughly 1 GB of the card, so the usable pool is about 7.0 GB (Legion) + about 6.5 GB (CachyOS, per llama.cpp's own free figure) ≈ 13.5 GB.

### RPC link between the machines

**Firewall (Legion, Administrator PowerShell):** the RPC worker has no authentication or encryption, so the rule admits only CachyOS on the Private profile (the direct cable):
```powershell
New-NetFirewallRule -DisplayName "llama RPC" -Direction Inbound -Protocol TCP -LocalPort 50052 -Action Allow -Profile Private -RemoteAddress 10.0.0.1
```

**Worker (Legion):** bound to the direct-link address only, not WiFi:
```powershell
cd D:\odysseus\llama.cpp
.\build\bin\Release\ggml-rpc-server.exe -H 10.0.0.2 -p 50052
```
Output: a warning banner ("Host ('10.0.0.2') is != '127.0.0.1' — never expose the RPC server to an open network") — expected for any non-loopback bind, and mitigated by the address binding plus the firewall rule. Then `Starting RPC server v7.0.0`, `endpoint : 10.0.0.2:50052`, `local cache : n/a` (the `-c` cache flag isn't used yet, so weights are re-sent on every client start), `CUDA0: NVIDIA GeForce RTX 5060 Laptop GPU (8123 MiB, 7043 MiB free)`, `transport : TCP`.

**Reachability checks from CachyOS:**
```fish
ping -c 3 10.0.0.2                                       # 3/3 replies, 0% loss, ~2.2 ms avg
bash -c 'timeout 3 bash -c "</dev/tcp/10.0.0.2/50052" && echo PORT OPEN || echo PORT CLOSED'    # PORT OPEN
```
The `bash -c` wrapper exists because `/dev/tcp` is a bash feature that fish doesn't have. Each connection the worker receives prints `Accepted client connection` / `Client connection closed` in its window; llama.cpp opens several short probe connections at startup to query devices and free memory, so multiple pairs of those lines are normal and don't mean the worker died.

**Unresolved oddity:** `llama-server --rpc 10.0.0.2:50052 --list-devices` printed only its `initializing ...` line and no device list on the *first* attempt. `llama-bench --rpc` worked fine right after, so the RPC path is OK; this turned out to be transient — a later retry (see section 5) printed the full device list correctly.

**First pooled run** (worker running on Legion, model on CachyOS's HDD):
```fish
./build/bin/llama-bench -m /mnt/1TB/odysseus-models/Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf -ngl 99 --rpc 10.0.0.2:50052
```
- The backend column read `CUDA,RPC`; the Legion's worker log showed `CUDA graph warmup complete`, so the 5060 really computed for CachyOS.
- **pp512 1222.76 ± 18.40 t/s, tg128 60.01 ± 0.04 t/s** — prompt processing about 14% below the single-GPU baseline (1415), generation identical to it (59.98).
- **Read this with care:** an 8B Q4 model fits on the 2060 Super alone, so pooling can't be expected to help, and the identical tg128 raised the question of how much of the model actually went to the Legion. Resolved in section 5.

## 5. Layer-split verification and tensor-split sweep (2026-09-25)

Goal: settle whether the "first pooled run" above actually split the model across both GPUs, and if so, whether the split ratio changes throughput.

**Device visibility check:**
```fish
./build/bin/llama-server --rpc 10.0.0.2:50052 --list-devices
```
```
Available devices:
  CUDA0: NVIDIA GeForce RTX 2060 SUPER (7798 MiB, 6438 MiB free)
  RPC0: 10.0.0.2:50052 (8123 MiB, 7029 MiB free)
```
Both devices are correctly enumerated through the RPC path. Also confirmed the binary is actually linking the RPC backend:
```fish
ls -l ./build/bin/ | grep -i rpc          # ggml-rpc-server, libggml-rpc.so(.0)(.25.0), test-rpc-multi-server present
ldd ./build/bin/llama-server | grep -i rpc  # libggml-rpc.so.0 => .../build/bin/libggml-rpc.so.0
```

**Tensor-split sweep, 8B Q4_K_M, `-ngl 99 --rpc 10.0.0.2:50052`:**

| `--tensor-split` (CUDA0,RPC0) | Result | tg (1000-token run) |
|---|---|---|
| `1,1` (even) | Loads clean | ~59.6–60.0 t/s (varies within-run) |
| `0.50,1` | **Fails** — CUDA0 OOM allocating ~4011 MiB KV cache buffer | — |
| `0.60,1` | **Fails** — CUDA0 OOM allocating ~3629 MiB | — |
| `0.65,1` | **Fails** — CUDA0 OOM allocating ~3629 MiB | — |
| `0.66,1` | Loads, but `compute buffer allocation failed, retrying without pipeline parallelism` | 59.61 t/s |
| `0.67,1` | Loads, same fallback warning; repeated twice | 59.69 t/s / 59.60 t/s |
| `0.68,1` | Loads, same fallback warning | 59.73 t/s (highest observed) |
| `0.69,1` | Loads, same fallback warning | 59.62 t/s |
| `0.70,1` | Loads clean, no fallback warning | 59.57 t/s |

Benchmark prompt used for the 1000-token runs: *"Write a very detailed explanation of distributed computing, GPU parallelism, model parallelism, pipeline parallelism, tensor parallelism, and why multiple GPUs can be useful for running large language models. Explain each concept carefully with examples."* (`n_predict: 1000`).

**Findings:**
- **The split is real.** `0.50`–`0.65` failing with a CUDA0 out-of-memory error while trying to allocate the KV cache buffer is direct proof CUDA0 (the 2060 Super) was being asked to hold a specific, non-trivial share of the model — a purely-remote setup wouldn't OOM the local card at all. `0.66` is the practical lower bound for CUDA0's share before the 2060 Super runs out of the ~6.4 GB llama.cpp reports free.
- **The split ratio barely moves throughput.** Every successful split — including the even `1,1` split — lands in the same ~58.4–60.0 t/s band, and that spread shows up *within* a single run (task-to-task) as much as it does *between* different split ratios. `0.68,1`'s 59.73 t/s is not a meaningful win over `0.67,1`'s 59.60–59.69 t/s.
- **Why:** the default split mode is `-sm layer` (pipelined) — each token's forward pass runs through CUDA0's layers, then RPC0's layers, in sequence. The two GPUs never compute simultaneously on the same token, so total tg tracks whichever GPU is slower per layer, not the sum of both cards' capacity. Shifting the ratio changes *which* GPU is closer to being the bottleneck, but with two cards of broadly similar per-layer speed on a model that already fits, it doesn't change the outcome much.
- **The `compute buffer allocation failed, retrying without pipeline parallelism` warning at 0.66–0.69** is a secondary symptom of the same VRAM ceiling: llama.cpp tries to reserve a pipelining compute buffer on CUDA0, can't fit it in the remaining headroom, and falls back to non-pipelined execution automatically. It didn't cost throughput here, but it's a sign this GPU is right at its limit for this split range.
- **Practical takeaway:** for a model that fits on one card, tensor-split tuning isn't worth the time — pick something that loads cleanly without the fallback warning (e.g. `1,1` or `0.70,1`) and move on. The real test of pooling is a model that requires the combined VRAM to run at all.

### Open items (inference layer)

1. **Pooled test with a model larger than one card's free VRAM** (~13–14B at Q4, about 8–9 GB), which can only run through the pool. This is the real test of whether pooling helps, since the 8B tests above were bottlenecked by the slower GPU per layer regardless of tensor-split ratio. Benchmark tok/s, GPU utilization, VRAM, and try `-sm row` or `-sm tensor` in addition to the default `layer` split, since parallel split modes may behave differently across a slow network link than the pipelined default. Consider `-c` on the worker to avoid re-sending weights on every restart.
2. Reboot check of the `10.0.0.1` static IP and an `iperf3` re-run (still unconfirmed).
3. Baseline of the same 8B model on the 5060 alone (Legion), for comparison.
4. Move the 1B smoke-test model from `~/odysseus/models` to `/mnt/1TB/odysseus-models`.
5. README status section needs updating to match this log.

## Benchmark results

**Network layer (complete):**

| Path | Avg throughput | Retransmits | Notes |
|---|---|---|---|
| Satellite (wired both ends) | ~94 Mbps | 0 (flat cap) | Fast Ethernet negotiation on satellite LAN port |
| Satellite bridge (CachyOS on main router, Legion on satellite) | ~154 Mbps | Frequent | Wireless backhaul instability |
| Direct WiFi (Legion, no satellite bridging) | ~224 Mbps | Frequent | Better than bridged, still unstable |
| **Direct cable (onboard-to-onboard)** | **942 Mbps** | **0** | Adopted — gigabit line-rate |
| USB adapter → main router (internet) | ISP-capped (~109/79 Mbps) | — | Adapter itself negotiates full gigabit via `ethtool` |

Direct link latency on 2026-09-24: ping 1.98 / 2.19 / 2.48 ms (min/avg/max), 0% loss.

**Inference layer (8B model fully characterized; larger-than-VRAM model still to test):**

| Test | Setup | Prompt (pp512) | Generation (tg128) | Notes |
|---|---|---|---|---|
| Single GPU, smoke test | RTX 2060 Super, Gemma 3 1B Q4_K_M, `llama-cli`, commit `957538960` | 252.5 t/s | 160.2 t/s | Tiny model and short prompt; pipeline check only |
| **Single GPU, baseline** | RTX 2060 Super, Llama 3.1 8B Q4_K_M (4.58 GiB), `llama-bench -ngl 99` | **1415.00 ± 13.77 t/s** | **59.98 ± 0.03 t/s** | Reference for pooled comparisons |
| Pooled via RPC, even split | Same 8B model, CachyOS client + Legion 5060 worker over the direct link, `--rpc 10.0.0.2:50052` | 1222.76 ± 18.40 t/s | 60.01 ± 0.04 t/s | Backend `CUDA,RPC`; split confirmed real (section 5), but model fits on one card so tg is unaffected |
| Pooled via RPC, split sweep 0.66–0.70 | Same 8B model, `--tensor-split` from 0.66,1 to 0.70,1 | not re-measured | 59.57–59.73 t/s | All splits within run-to-run noise; see section 5 table |

Both machines are built on commit `957538960` and the RPC link works end to end, with the layer split now confirmed. Still to do: a model larger than one card's VRAM — the real test of pooling.

## Command reference — everything run during network setup, debugging and the build

### Installing iperf3

```bash
# CachyOS
sudo pacman -S iperf3
```
```powershell
# Legion — winget install didn't add it to PATH automatically
winget install iperf3
winget list iperf3   # confirm install + find version
# Find the actual binary if `iperf3` isn't recognized:
Get-ChildItem -Path "C:\Users\$env:USERNAME\AppData\Local\Microsoft\WinGet\Packages" -Recurse -Filter "iperf3.exe"
# Run directly via full path, or make it permanent:
$env:Path += ";C:\Users\CY4NAD3\AppData\Local\Microsoft\WinGet\Packages\ar51an.iPerf3_Microsoft.Winget.Source_8wekyb3d8bbwe"
[Environment]::SetEnvironmentVariable("Path", $env:Path, "User")
```

### iperf3 testing

```bash
# Server (Legion side, run first)
iperf3 -s

# Client (CachyOS side) — -c = client mode, -t = duration in seconds
iperf3 -c <legion_ip> -t 30
iperf3 -c 10.0.0.2 -t 30      # over the direct link
iperf3 --version
iperf3 --help
```

### Basic connectivity / interface inspection

```bash
ip a                            # list all interfaces + IPs
ip a show enp3s0                # inspect one interface
ip route                        # show routing table / default gateway
ping -c 4 <ip>                  # -c = packet count, then auto-stop
sudo ip neigh flush all         # clear ARP cache
arp -a | grep <ip>              # check if an IP resolved to a MAC (ARP)
```

```powershell
ipconfig                        # Windows equivalent of `ip a`
ping <ip> -n 20                 # Windows ping, -n = count (not -c)
Get-NetAdapter                  # list adapters + status
Get-NetAdapter | Select-Object Name, LinkSpeed
Test-NetConnection -ComputerName <ip> -InformationLevel Detailed
```

### Link speed / negotiation

```bash
sudo ethtool enp3s0             # check Speed, Duplex, Link detected
sudo ethtool enp1s0f0u1
```

### USB device / chipset identification

```bash
lsusb                           # list USB devices, confirm chipset (e.g. Realtek RTL8153 = 0bda:8153)
```

### Static IP assignment (direct point-to-point link)

```bash
sudo ip addr add 10.0.0.1/24 dev enp3s0
sudo ip link set enp3s0 up
sudo ip addr del 10.0.0.1/24 dev enp3s0     # to remove it again
```
```powershell
netsh interface ipv4 set address name="Ethernet" static 10.0.0.2 255.255.255.0
```

### NetworkManager (CachyOS)

```bash
nmcli device status                          # see all interfaces + connection state
nmcli connection down "Wired connection 2"
nmcli connection up "Wired connection 2"
sudo systemctl restart NetworkManager        # force a fresh DHCP attempt
```

### Windows network profile / firewall (the recurring root cause)

```powershell
Get-NetConnectionProfile                                    # check Public vs Private
Set-NetConnectionProfile -InterfaceAlias "Ethernet" -NetworkCategory Private
Get-NetFirewallProfile | Select-Object Name, Enabled
Set-NetFirewallProfile -Profile Private -Enabled False       # temporary test
Set-NetFirewallProfile -All -Enabled False                   # temporary test, all profiles
Set-NetFirewallProfile -All -Enabled True                    # re-enable after testing
New-NetFirewallRule -DisplayName "iperf3" -Direction Inbound -Protocol TCP -LocalPort 5201 -Action Allow
New-NetFirewallRule -DisplayName "Allow ICMPv4-In" -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow
# RPC worker port, restricted to the CachyOS end of the direct link (run as Administrator):
New-NetFirewallRule -DisplayName "llama RPC" -Direction Inbound -Protocol TCP -LocalPort 50052 -Action Allow -Profile Private -RemoteAddress 10.0.0.1
```

### ufw (CachyOS firewall) — ruled out as the cause, but checked

```bash
sudo ufw status verbose
sudo journalctl -k --since "5 minutes ago" | grep -i "UFW BLOCK"   # kernel-level block log via journald (no /var/log/ufw.log on this system)
```

### ISP-side throughput check (for context, not the RPC link)

```bash
sudo pacman -S speedtest-cli
speedtest-cli
```

### Storage / mounts (CachyOS)

```bash
df -h ~                                                  # free space on /home
lsblk -f                                                 # drives, filesystems, labels, UUIDs, mountpoints
findmnt -o TARGET,SOURCE,FSTYPE,OPTIONS /mnt/1TB          # confirm ntfs3 + rw + options
mount | grep "1TB"                                       # same check, shorter: look for rw and uid=1000
cat /etc/fstab                                           # auto-mount entries (ntfs3, nofail, uid/gid)
ls /mnt/1TB                                              # same files Dolphin shows under "1 TB Drive"
```

### Models: download, locate, copy, move

```bash
mkdir -p /mnt/1TB/odysseus-models                         # -p = no error if it already exists
cd /mnt/1TB/odysseus-models
wget -c <direct .gguf URL>                               # -c = resume a broken download
ls -lh /mnt/1TB/odysseus-models/                          # -h = human-readable sizes (sanity-check the size)
find ~/.cache -iname "*gemma-3-1b*" 2>/dev/null           # where did -hf put it?
cp -L <symlink path> <destination>/                      # -L = follow the symlink and copy the real file
mv ~/odysseus/models/gemma-3-1b-it-Q4_K_M.gguf /mnt/1TB/odysseus-models/   # across drives = copy, then delete original
```

### CUDA / build toolchain checks (CachyOS)

```bash
pacman -Q cuda nvidia-utils 2>/dev/null   # is the toolkit package installed?
ls /opt/cuda/bin/nvcc                     # the compiler llama.cpp needs
/opt/cuda/bin/nvcc --version              # which CUDA version
nvidia-smi                                # driver working + current VRAM usage
fish_add_path /opt/cuda/bin               # fish-shell equivalent of `export PATH=...` (here: already on PATH)
```

### llama.cpp build and verification (CachyOS; the Legion version is in section 4)

```bash
git clone https://github.com/ggml-org/llama.cpp && cd llama.cpp
git rev-parse --short HEAD                # note the hash — must match on both machines
cmake -B build -DGGML_CUDA=ON -DGGML_RPC=ON -DCMAKE_CUDA_ARCHITECTURES=75   # 75 = CachyOS/2060S, 120 = Legion/5060
cmake --build build --config Release -j6
./build/bin/llama-server --version        # version, build number, commit
ls build/bin | grep -E "rpc|server|cli"   # the RPC worker is `ggml-rpc-server` in this version
ls build/bin | grep -i cuda               # libggml-cuda.so = the GPU backend was built
./build/bin/ggml-rpc-server --help        # host/port/cache options (default host 127.0.0.1, port 50052)
./build/bin/llama-server --list-devices   # proof llama.cpp can see the GPU and its free VRAM
./build/bin/llama-cli -hf ggml-org/gemma-3-1b-it-GGUF:Q4_K_M -ngl 99 -p "..."   # smoke test, downloads to the HF cache
./build/bin/llama-cli -m <model.gguf> -ngl 99 -p "..."                          # same, from a file you chose
mkdir -p ~/odysseus/models                # local model folder
```

### Benchmarking (`llama-bench`)

```bash
# single GPU baseline — pp512 = prompt processing, tg128 = text generation
./build/bin/llama-bench -m /mnt/1TB/odysseus-models/Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf -ngl 99
# pooled: same command plus the remote worker
./build/bin/llama-bench -m /mnt/1TB/odysseus-models/Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf -ngl 99 --rpc 10.0.0.2:50052
```
- `-ngl 99` = offload up to 99 layers to the GPU(s), i.e. everything.
- The `--rpc` run prints `CUDA,RPC` in the backend column when the remote device is in use.

### Layer-split verification and tensor-split sweep (section 5)

```fish
# confirm both devices are visible through the RPC path
./build/bin/llama-server --rpc 10.0.0.2:50052 --list-devices

# confirm the binary actually links the RPC backend
ls -l ./build/bin/ | grep -i rpc
ldd ./build/bin/llama-server | grep -i rpc

# run the server with a specific split and watch it load
./build/bin/llama-server \
    -m /mnt/1TB/odysseus-models/Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf \
    -ngl 99 \
    --rpc 10.0.0.2:50052 \
    --tensor-split 0.67,1        # fraction of the model on CUDA0,RPC0 — not a percentage, a ratio
```
- `--tensor-split N0,N1,...` sets each device's share of the model as a ratio, one number per device in the order `--list-devices` reports them (here CUDA0 then RPC0).
- `-fit`/`--fit` (auto-fit) is what `-ngl 99` combined with an explicit split disables — hence the harmless `failed to fit params to free device memory: n_gpu_layers already set by user to 99, abort` line on every run in this sweep; it just means llama.cpp isn't auto-picking layer counts because they were already forced.
- A CUDA0 `cudaMalloc failed: out of memory` while allocating the KV cache buffer means the split is asking the local GPU to hold more than its free VRAM — lower CUDA0's share.
- `compute buffer allocation failed, retrying without pipeline parallelism` is llama.cpp falling back automatically when it can't also fit the small pipelining buffer — informational, not fatal.

### Windows MSVC toolchain and CUDA (Legion)

```powershell
# does a real C++ compiler install exist? (plain `-latest` also returns non-C++ products such as SSMS)
& "${env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\vswhere.exe" -products * -latest -property installationPath
& "${env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\vswhere.exe" -products * -requires Microsoft.VisualStudio.Component.VC.Tools.x86.x64 -property installationPath
dir "C:\Program Files\Microsoft Visual Studio\18\Community\VC\Tools\MSVC"                       # check existing (VS2026) toolset
winget install Microsoft.VisualStudio.2022.BuildTools --override "--passive --wait --add Microsoft.VisualStudio.Workload.VCTools --includeRecommended"
dir "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Tools\MSVC"               # confirm v143 toolset landed
winget install Kitware.CMake
Test-Path "C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA"                                   # is the CUDA Toolkit installed?
nvcc --version                                                                                   # new PowerShell window after installing
cmake --version
Get-ChildItem "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\MSBuild\Microsoft\VC\v170\BuildCustomizations" -Filter "CUDA*"   # did the VS integration attach?
```

### RPC pooling (worker on Legion, client on CachyOS)

```powershell
# Legion — start the worker, bound to the direct-link address only
cd D:\odysseus\llama.cpp
.\build\bin\Release\ggml-rpc-server.exe -H 10.0.0.2 -p 50052
```
```fish
# CachyOS — reachability, then the pooled benchmark
ping -c 3 10.0.0.2
bash -c 'timeout 3 bash -c "</dev/tcp/10.0.0.2/50052" && echo PORT OPEN || echo PORT CLOSED'
pgrep -a llama                                            # any leftover llama process holding a connection?
```

## Key learnings

- **Benchmark before building**: running `iperf3` early caught the satellite bottleneck before any time was sunk into llama.cpp setup on a network that couldn't support it well.
- **Silent failure modes are dangerous**: Windows' "Public" network profile silently blocks ICMP/TCP with no obvious error — hit this twice, on two different interfaces. Always check `Get-NetConnectionProfile` first on unexpected Windows connectivity failures.
- **A flat, suspiciously round throughput number is a clue**: the ~94 Mbps flat result wasn't "slow wifi," it was 100Mb/s Fast Ethernet negotiation — physical-layer issues can look like higher-level ones.
- **Chipset consistency matters more than price**: RTL8153 was chosen specifically for in-kernel Linux driver support and native Windows support — paid a premium over unbranded alternatives deliberately, validated by the 942 Mbps/zero-retransmit result.
- **Direct connections beat clever routing**: 942 Mbps over a direct cable vs. ~154–224 Mbps over any form of the mesh — minimizing hops and shared media wins for latency-sensitive workloads like RPC.
- **Protect fragile physical ports**: decided to offload direct-link cable duty from Legion's hinge-mounted Ethernet port onto a USB adapter, rather than risk wearing it out with repeated connect/disconnect cycles.
- **Live state ≠ persisted state**: `sudo ip addr add` sets an address immediately but doesn't survive a NetworkManager restart or reboot — needed a proper `nmcli` connection profile, and that kind of fix should always be reboot-tested before being marked done. A working `ping` after the fact is not that test.
- **A prior, unrelated install can save a step**: CUDA was already fully installed on CachyOS from earlier hashcat work, not just the driver — worth checking what's already there (`pacman -Q`, `nvcc --version`) before assuming a clean install is needed.
- **Shell-specific syntax breaks silently**: bash's `export PATH=...` under fish mangles PATH for that session (core commands like `uname` and `id` stop resolving); fish needs `fish_add_path`. Check `$SHELL` before pasting PATH-setting commands from generic instructions. Same family: bash's `/dev/tcp` trick needs a `bash -c` wrapper under fish.
- **A newer toolset isn't automatically compatible**: Visual Studio 2026 shipping only the v145 MSVC toolset (no v143) meant `nvcc` had nothing validated to compile against, even though *a* compiler was technically present — CUDA toolkit/compiler version support lags behind the newest IDE toolsets.
- **Unrelated toolchains can coexist safely if you're explicit**: Code::Blocks/MinGW staying on the Legion for GLUT coursework doesn't conflict with the CUDA/MSVC build, as long as the compiler is named explicitly (`-G "Visual Studio 17 2022"`) rather than left for CMake to infer from `PATH`.
- **Verify names against the actual build output**: the RPC worker is `ggml-rpc-server` in this llama.cpp version, not `rpc-server`. Program names and flags change between versions, so `ls build/bin` and `--help` beat memory — for example, `-no-cnv` is rejected as an invalid argument in this build.
- **`<placeholders>` in example commands must be replaced entirely, brackets included**: the shell reads `<` as input redirection, so pasting `<file>` literally fails with a confusing "path does not exist" error.
- **`vswhere -latest` isn't "the latest C++ install"**: it returns any Visual Studio–family product (it returned SQL Server Management Studio). Filter with `-requires Microsoft.VisualStudio.Component.VC.Tools.x86.x64`.
- **Two different free-VRAM numbers**: `nvidia-smi` and llama.cpp report different totals (8192 MiB vs 7798 MiB on the 2060 Super). Use llama.cpp's figure when sizing models. On the Legion, Windows takes about 1 GB of the 5060 (8123 MiB total, 7043 MiB free).
- **Pooling only pays off when the model doesn't fit on one card**: below that size, every token crossing the cable makes pooled inference slower than a single GPU, so pooled tests need a model larger than one card's free VRAM. And RPC has no authentication or encryption, so the worker should bind only to the private direct-link address and be firewalled to the client.
- **Pin the llama.cpp commit on both machines**: the RPC protocol is version-sensitive, so record `git rev-parse --short HEAD` on the first machine and `git checkout` it on the second.
- **`-hf` hides where the model went**: it downloads into the Hugging Face cache, stored as a real file in `blobs/` plus a nicely named symlink in `snapshots/`. Copy with `cp -L` (a plain `mv` moves only the link). Better: download `.gguf` files yourself with `wget -c` into a folder you chose and load them with `-m`.
- **The HDD only costs load time, not speed**: once the weights are in VRAM the disk isn't touched, so tokens/s is identical from an HDD or an SSD; only the first load (about 35–50 s for a 5 GB file) is slower. The internet line (~13 MB/s), not the HDD, limits downloads.
- **Volume label ≠ mount point**: Dolphin shows the label ("1 TB Drive"), the terminal uses the mount path (`/mnt/1TB`). Read the real path from `lsblk -f` or `mount` rather than guessing — a path with a space in it also needs quotes.
- **Open a fresh terminal after installing tools on Windows**: a running PowerShell keeps the PATH it started with, so a successful install can still look like "command not recognized".
- **Installer summaries can look worse than they are**: the "Nsight for VS 2022 not installed" line was irrelevant; the meaningful check was that the `CUDA 13.4.*` files appeared in the Build Tools' `BuildCustomizations` folder.
- **Don't trust one number — check what actually happened**: the pooled 8B run showing tg128 identical to the single-GPU baseline (60.01 vs 59.98) doesn't by itself prove the model was split. `--list-devices`, `ldd`, and deliberately forcing an OOM at a low `--tensor-split` ratio are what actually proved it (section 5).
- **`llama-cli` opens an interactive chat and blocks scripted runs**: piping its output to `grep` or feeding it empty input just hangs at the `>` prompt, and the log stays empty. For startup-log checks use `llama-server` (no chat prompt) or `llama-bench`.
- **Worker log noise is normal**: repeated `Accepted client connection` / `Client connection closed` lines are the short probe connections llama.cpp makes when it enumerates devices; the worker only exits if the process itself stops (the `PS` prompt comes back).
- **Auto-fit and manual layer counts don't mix**: setting `-ngl 99` explicitly disables llama.cpp's automatic VRAM-fitting logic, so every run in the tensor-split sweep printed `failed to fit params to free device memory: n_gpu_layers already set by user to 99, abort` — expected noise, not an error, once you're deliberately overriding the layer count.
- **The pipelined default split mode caps pooled speed at the slower GPU's per-layer rate**: with `-sm layer` (the default), tg128 for a model that fits on one card barely moves across tensor-split ratios (58.4–60.0 t/s the whole way from `0.66,1` to `1,1`), because the two GPUs run in relay rather than in parallel. Ratio tuning only matters for fitting a bigger model in, not for speed, on this split mode — `-sm row` or `-sm tensor` are the modes to test for actual parallel throughput.
- **A CUDA OOM at a specific tensor-split ratio is proof of a real split, and useful proof**: deliberately pushing a split ratio (`0.50,1` → `0.65,1`) until CUDA0 fails to allocate the KV cache buffer confirmed the local GPU really was being assigned that fraction of the model — a much more direct test than reading throughput numbers, and it also mapped the exact VRAM ceiling (`0.66,1` is the practical floor for CUDA0's share on this model).
