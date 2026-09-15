# Build Log — Project Odysseus

The full debugging story behind the network layer — including the dead ends and every command used. See [README.md](./README.md) for the project overview and current status.

## Hardware

**CachyOS Desktop** (main node)
- CPU: Ryzen 5 2600X · GPU: RTX 2060 Super 8GB · RAM: 16GB
- Username `azraf`, hostname `AzureLinux`

**Legion 5** (RPC worker)
- CPU: Ryzen 7 · GPU: RTX 5060 8GB · RAM: 16GB · OS: Windows, username `CY4NAD3`

Note: the two GPUs are different generations (Turing vs. Ada/Blackwell-class) — compute capability mismatch is a variable to watch during inference benchmarking, since the cards won't process synchronized layers at the same rate.

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

## Open item

CachyOS's `10.0.0.1` static IP on `enp3s0` is currently set via `sudo ip addr add 10.0.0.1/24 dev enp3s0` — **temporary**, wiped on NetworkManager restarts/reboots. Needs a persistent NetworkManager connection profile (`nmcli con add` with a fixed IP) before this is considered done.

## Next up

Buying a **second RTL8153-based USB-Ethernet adapter for Legion**, sourced in person through an ISP technician contact rather than an unbranded online listing. This will take over the direct RPC cable on Legion's end, freeing Legion's onboard Ethernet port (a fragile hinge-mounted connector) from repeated plug/unplug wear.

## Benchmark results

**Network layer (complete):**

| Path | Avg throughput | Retransmits | Notes |
|---|---|---|---|
| Satellite (wired both ends) | ~94 Mbps | 0 (flat cap) | Fast Ethernet negotiation on satellite LAN port |
| Satellite bridge (CachyOS on main router, Legion on satellite) | ~154 Mbps | Frequent | Wireless backhaul instability |
| Direct WiFi (Legion, no satellite bridging) | ~224 Mbps | Frequent | Better than bridged, still unstable |
| **Direct cable (onboard-to-onboard)** | **942 Mbps** | **0** | Adopted — gigabit line-rate |
| USB adapter → main router (internet) | ISP-capped (~109/79 Mbps) | — | Adapter itself negotiates full gigabit via `ethtool` |

**Inference layer:** pending — llama.cpp build and RPC pooling not yet started.

## Command reference — everything run during network setup/debugging

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

## Key learnings

- **Benchmark before building**: running `iperf3` early caught the satellite bottleneck before any time was sunk into llama.cpp setup on a network that couldn't support it well.
- **Silent failure modes are dangerous**: Windows' "Public" network profile silently blocks ICMP/TCP with no obvious error — hit this twice, on two different interfaces. Always check `Get-NetConnectionProfile` first on unexpected Windows connectivity failures.
- **A flat, suspiciously round throughput number is a clue**: the ~94 Mbps flat result wasn't "slow wifi," it was 100Mb/s Fast Ethernet negotiation — physical-layer issues can look like higher-level ones.
- **Chipset consistency matters more than price**: RTL8153 was chosen specifically for in-kernel Linux driver support and native Windows support — paid a premium over unbranded alternatives deliberately, validated by the 942 Mbps/zero-retransmit result.
- **Direct connections beat clever routing**: 942 Mbps over a direct cable vs. ~154–224 Mbps over any form of the mesh — minimizing hops and shared media wins for latency-sensitive workloads like RPC.
- **Protect fragile physical ports**: decided to offload direct-link cable duty from Legion's hinge-mounted Ethernet port onto a USB adapter, rather than risk wearing it out with repeated connect/disconnect cycles.
