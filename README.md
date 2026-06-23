# Private 5G SA Network (srsRAN + Open5GS)

![Platform](https://img.shields.io/badge/platform-Ubuntu%2024.04%20LTS-orange)
![Radio](https://img.shields.io/badge/SDR-USRP%20B210-blue)
![Core](https://img.shields.io/badge/core-Open5GS%20v2.7.6-green)
![RAN](https://img.shields.io/badge/gNB-srsRAN%20Project%2025.10.0-green)
![UE](https://img.shields.io/badge/UE-srsRAN__4G-green)
![Status](https://img.shields.io/badge/status-working%20end--to--end-success)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

A complete, real, end-to-end build log for a **private 5G Standalone (SA) network**, built entirely with open-source software and software-defined radio — no vendor hardware, no simulators.

This repo documents **every command run, every error hit, and the exact fix that resolved it**, in the order it actually happened. If you're trying to bring up srsRAN + Open5GS on real SDR hardware and you're stuck, there's a good chance the exact error you're seeing is in here, with the root cause and fix.

> **Result:** A fully working 5G SA call — RF cell search, SFN sync, SIB1 acquisition, Random Access, RRC Connection, NAS Authentication, Security Mode, and active PDSCH/PUSCH data transfer at 64QAM — established end-to-end over the air between two USRP B210 radios.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Network Identifiers](#network-identifiers)
- [Quick Start](#quick-start)
- [Repo Structure](#repo-structure)
- [Phase 1 — Open5GS Core Network Setup (PC2)](#phase-1--open5gs-core-network-setup-pc2)
- [Pre-Flight Check — Verifying a Clean Environment](#pre-flight-check--verifying-a-clean-environment)
- [Phase 3 — Open5GS WebUI Login Failure](#phase-3--open5gs-webui-login-failure)
- [Phase 4 — Building srsRAN Project (gNB) on PC2](#phase-4--building-srsran-project-gnb-on-pc2)
- [Phase 5 — gNB Configuration File Debugging](#phase-5--gnb-configuration-file-debugging)
- [Phase 6 — Building and Configuring srsUE on PC1](#phase-6--building-and-configuring-srsue-on-pc1)
- [Phase 7 — First Successful End-to-End 5G SA Connection](#phase-7--first-successful-end-to-end-5g-sa-connection)
- [Phase 8 — Clock Drift Issue on Reconnection](#phase-8--clock-drift-issue-on-reconnection)
- [Appendix — Quick Command Reference](#appendix--quick-command-reference)
- [License](#license)

---

## Overview

**Goal:** deploy a working 5G SA network across two PCs:

| Machine | Role |
|---|---|
| **PC2** | Open5GS (5G Core Network) + srsRAN Project (gNB / Radio Access Network) |
| **PC1** | srsUE — srsRAN 4G's 5G NR-capable UE application |

Each PC drives its own USRP B210, and the two communicate over the air (or via cable).

**Stack:**

| Component | Software | Version |
|---|---|---|
| Platform | Ubuntu | 24.04 LTS |
| SDR | Ettus USRP B210 | ×2 |
| 5G Core | Open5GS | v2.7.6 |
| gNB (RAN) | srsRAN Project | 25.10.0 |
| UE | srsRAN_4G (srsue)     | 25.10.0 |

## Architecture

```
┌──────────────────────────────┐        ┌──────────────────────────────┐
│             PC1               │        │             PC2               │
│       Ubuntu 24.04 LTS         │        │       Ubuntu 24.04 LTS         │
│                                │        │                                │
│  ┌──────────────────────┐      │        │  ┌──────────────────────┐      │
│  │   srsUE (srsRAN 4G)   │      │        │  │  srsRAN Project gNB   │      │
│  │   5G NR UE app        │      │  RF    │  │  (CU/DU + Lower PHY)  │      │
│  └──────────┬───────────┘      │  over  │  └──────────┬───────────┘      │
│             │ UHD               │  air / │             │ UHD               │
│  ┌──────────▼───────────┐      │  cable │  ┌──────────▼───────────┐      │
│  │   USRP B210 #1        │◄─────┼────────┼─►│   USRP B210 #2        │      │
│  └────────────────────────┘    │        │  └────────────────────────┘    │
│                                │        │             │ N2 / NGAP (SCTP)  │
│                                │        │  ┌──────────▼───────────┐      │
│                                │        │  │   Open5GS 5G Core      │      │
│                                │        │  │ AMF / SMF / UPF / NRF / │      │
│                                │        │  │     AUSF / UDM / UDR    │      │
│                                │        │  └────────────────────────┘    │
└──────────────────────────────┘        └──────────────────────────────┘
```

## Network Identifiers

| Parameter | Value |
|---|---|
| PLMN (MCC/MNC) | 001 / 01 |
| TAC | 7 |
| SST | 1 |
| Band | n3 (1842.5 MHz DL / 1747.5 MHz UL) |
| Bandwidth | 20 MHz |
| IMSI | `001010123456788` |
| Subscriber Key (K) | `00112233445566778899aabbccddeeff` |
| OPc | `63BFA50EE6523365FF14C1F45F88737D` |
| APN / DNN | `srsapn` |

> The IMSI above is just a lab identifier — any valid value works as long as it matches **exactly** between the Open5GS subscriber record and the `ue.conf` `[usim]` block.

## Quick Start

If you just want the commands without the debugging story, jump straight to the [Appendix — Quick Command Reference](#appendix--quick-command-reference). Otherwise, read on — each phase below explains *why* things failed the way they did, so you can recognize the same failure if you hit it.

Ready-to-copy config files (gNB YAML, UE config, systemd unit) live in [`configs/`](./configs) — no need to retype anything from the code blocks below.

## Repo Structure

```
.
├── README.md                   # this file — full build log, issues, and fixes
├── LICENSE
├── configs/
│   ├── README.md                # what each config file is and how to adapt it
│   ├── gnb_srsRAN.yml            # srsRAN Project gNB config (PC2)
│   ├── ue.conf                   # srsUE config (PC1)
│   └── open5gs-webui.service     # systemd unit for the Open5GS WebUI (PC2)
```

---

## Phase 1 — Open5GS Core Network Setup (PC2)

### 1.1 Install prerequisites

```bash
sudo apt update && sudo apt upgrade -y

sudo apt-get install -y cmake make gcc g++ pkg-config \
  libfftw3-dev libmbedtls-dev libsctp-dev libyaml-cpp-dev \
  libgtest-dev build-essential libboost-program-options-dev \
  libconfig++-dev git curl gnupg apt-transport-https
```

### 1.2 Install UHD (USRP Hardware Driver)

```bash
sudo add-apt-repository ppa:ettusresearch/uhd
sudo apt-get update
sudo apt-get install -y libuhd-dev uhd-host
sudo /usr/libexec/uhd/utils/uhd_images_downloader.py

# Verify B210 detection (USB 3.0 port required)
uhd_usrp_probe
```

### 1.3 Install Open5GS Core

```bash
sudo add-apt-repository ppa:open5gs/latest
sudo apt update
sudo apt install -y open5gs

# MongoDB (subscriber database)
curl -fsSL https://pgp.mongodb.com/server-8.0.asc | \
  sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] \
  https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list
sudo apt update
sudo apt install -y mongodb-org
sudo systemctl start mongod
sudo systemctl enable mongod
```

✅ **Result:** Open5GS installed cleanly via PPA at version `2.8.0~noble2`. All core network functions — AMF, SMF, UPF, NRF, AUSF, UDM, UDR, PCF, BSF, NSSF, SCP, SEPP, HSS, MME, SGWC, SGWU — came up as systemd services.

### 1.4 Install the Open5GS WebUI

The official one-line installer was tried first:

```bash
curl -fsSL https://open5gs.org/open5gs/assets/webui/install | sudo -E bash -
```

> ⚠️ **Issue:** The installer failed downloading the source tarball:
> ```
> + curl -sLf 'https://github.com/open5gs/open5gs/archive/v2.8.0.tar.gz' | tar zxf -
> gzip: stdin: unexpected end of file
> tar: Child returned status 1
> ```

**Diagnosis:** the installer hardcodes a download URL for git tag `v2.8.0` — but that tag doesn't actually exist on GitHub. `2.8.0` was only ever an apt/PPA package version, never a tagged GitHub release. Confirmed with a direct `wget`, which returned `404 Not Found` on the same URL.

✅ **Fix:** identified the real latest tagged release (`v2.7.6`) on the [official open5gs/open5gs repo](https://github.com/open5gs/open5gs) and cloned that directly instead of trusting the installer script.

```bash
cd ~
git clone --depth 1 --branch v2.7.6 https://github.com/open5gs/open5gs.git
cd open5gs/webui
npm install
npm run build
```

### 1.5 Run the WebUI as a systemd service

First attempt — copying build output into a separate directory:

```bash
sudo mkdir -p /usr/lib/node_modules/open5gs
sudo cp -r build /usr/lib/node_modules/open5gs/
sudo cp package.json /usr/lib/node_modules/open5gs/
sudo cp -r node_modules /usr/lib/node_modules/open5gs/
```

> ⚠️ **Issue:** service started, then immediately crashed:
> ```
> Error: Cannot find module '/usr/lib/node_modules/open5gs/server/index.js'
> code: 'MODULE_NOT_FOUND'
> ```

**Diagnosis:** this is a Next.js app. The `cp` step had silently failed earlier (`cp: cannot stat 'build': No such file or directory`) because the build hadn't finished producing output at copy time. Next.js apps are meant to run via `npm start` from their own project root (using their own `node_modules` and `package.json` together) — not by manually relocating loose files into a different directory.

✅ **Fix:** pointed systemd directly at the cloned project's working directory instead of copying files anywhere:

```bash
sudo tee /lib/systemd/system/open5gs-webui.service > /dev/null <<'EOF'
[Unit]
Description=Open5GS WebUI
After=mongod.service

[Service]
Type=simple
WorkingDirectory=/home/<your-user>/open5gs/webui
Environment=HOSTNAME=0.0.0.0
Environment=PORT=9999
ExecStart=/usr/bin/npm start
Restart=on-failure
RestartSec=5
User=<your-user>

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable open5gs-webui
sudo systemctl start open5gs-webui
```

✅ **Result:** `> Ready on http://0.0.0.0:9999` — WebUI reachable at `http://localhost:9999`.

---

## Pre-Flight Check — Verifying a Clean Environment

Before bringing up Open5GS, it's worth spending five minutes confirming nothing else on the box is already using the ports Open5GS needs. Skipping this leads to a confusing failure mode: a core network function (most often the AMF) starts, exits immediately, and crash-loops — even though its config file is completely correct.

### Why this happens

Open5GS's network functions bind to a fixed set of well-known ports on loopback:

| Port | Protocol | Used by |
|---|---|---|
| 38412 | SCTP | AMF — N2/NGAP interface to the gNB |
| 8805 | UDP | SMF/UPF — PFCP session interface |
| 2152 | UDP | UPF — GTP-U user-plane tunnel |
| 7777 | TCP | NRF/SBI — service-based interface used by all NFs |

If anything else is already bound to one of these — another 5G core (OAI, free5GC), a leftover Docker container, or a previous Open5GS process that didn't shut down cleanly — the corresponding daemon fails to bind and crashes with a generic `EXCEPTION` exit code that gives no direct clue about the real cause.

### Quick checks before starting

```bash
# Confirm no leftover containers are running
docker ps

# Confirm the ports are actually free (expect: no output)
sudo ss -lnp | grep -E '38412|8805|2152|7777'
```

If something unexpected shows up:

```bash
docker ps   # find the container name
docker stop <container_name>
docker rm <container_name>
```

Then start Open5GS and verify each service:

```bash
sudo systemctl restart open5gs-amfd open5gs-smfd open5gs-upfd open5gs-nrfd \
  open5gs-ausfd open5gs-udmd open5gs-udrd

sudo systemctl status open5gs-amfd
# Active: active (running)   <- healthy start
sudo ss -lnp | grep 38412
```

✅ **Result:** on a clean install with nothing else competing for these ports, every Open5GS daemon starts on the first try and registers with the NRF (`NF registered [Heartbeat:10s]` in the logs).

### If you hit this anyway

```bash
docker inspect <container> --format='{{ index .Config.Labels "com.docker.compose.project.working_dir"}}'
```
This recovers the original `docker-compose` project folder for a running container — useful if you need to bring that stack back up later.

---

## Phase 3 — Open5GS WebUI Login Failure

With the WebUI running on port 9999 and the core healthy, login with the documented default credentials (`admin` / `1423`) was attempted and rejected — for every combination tried (`admin`/`1423`, `admin`/`admin`, `admin`/`password`).

### Investigation

```bash
# Checked the MongoDB accounts collection — it was empty
grep -r "bcrypt\|password\|hash\|salt" server/ lib/ --include="*.js" -l
grep -r "bcrypt\|password\|1423" ~/open5gs/webui/.next/ | grep -v Binary
# -> Account/Document.js uses crypto.pbkdf2(password, salt, 25000, 512, 'sha256', ...)
```

**Root cause #1:** the WebUI does **not** use bcrypt — it uses Node's built-in `pbkdf2` (25,000 iterations, SHA-256, 512-byte derived key) via `passport-local-mongoose`. A manually-crafted bcrypt hash could never match.

Reading `server/index.js` directly exposed the real mechanism:

```js
if (dev) {
  Account.count((err, count) => {
    if (!count) {
      const newAccount = new Account();
      newAccount.username = 'admin';
      newAccount.roles = [ 'admin' ];
      Account.register(newAccount, '1423', err => { ... })
    }
  })
}
```

**Root cause #2 (the real one):** the default `admin`/`1423` account is **only** auto-created when the server runs with `NODE_ENV` in development mode. The systemd service explicitly set `NODE_ENV=production` for performance — so this bootstrap code never ran, and the `accounts` collection stayed empty forever.

✅ **Fix:** used the WebUI's own bundled `passport-local-mongoose` library — the exact function the app itself calls — to create the admin account correctly with a properly-derived pbkdf2 hash and salt, bypassing the dev-only bootstrap entirely:

```bash
cd ~/open5gs/webui

node -e "
const mongoose = require('./node_modules/mongoose');
const Account = require('./server/models/account.js');

mongoose.connect('mongodb://localhost/open5gs').then(() => {
  Account.countDocuments().then(count => {
    if (count > 0) { return Account.deleteMany({}); }
  }).then(() => {
    const newAccount = new Account();
    newAccount.username = 'admin';
    newAccount.roles = ['admin'];
    Account.register(newAccount, '1423', (err) => {
      console.log(err ? err : 'Admin account created with password: 1423');
      mongoose.disconnect();
    });
  });
});
"

sudo systemctl restart open5gs-webui
```

✅ **Result:** login succeeded at `http://localhost:9999` with `admin` / `1423`.

### Subscriber provisioning

Once logged in, a subscriber was added via **Subscribers → +** with the identifiers kept consistent across Open5GS and the srsUE config:

| Field | Value |
|---|---|
| IMSI | `001010123456788` |
| Subscriber Key (K) | `00112233445566778899aabbccddeeff` |
| OPc | `63BFA50EE6523365FF14C1F45F88737D` |
| AMF value | `8000` |
| APN / DNN | `srsapn` |
| SST | `1` |

> Note: the IMSI originally planned ended in `...780`, but the one actually provisioned ended in `...788`. Since the IMSI is just an identifier, this is fine — what matters is that **both sides match exactly**.

---

## Phase 4 — Building srsRAN Project (gNB) on PC2

The srsRAN Project gNB (the RAN component — CU/DU + lower PHY) had no working PPA package for Ubuntu 24.04 (Noble), so it was built from source.

### PPA attempt — failed

```bash
sudo add-apt-repository ppa:softwareradiosystems/srsran-project
sudo apt-get update
sudo apt-get install -y srsran-project
```

> ⚠️ **Issue:**
> ```
> Err:10 https://ppa.launchpadcontent.net/softwareradiosystems/srsran-project/ubuntu noble Release
>   404  Not Found
> E: Unable to locate package srsran-project
> ```

✅ **Fix:** removed the broken PPA and built from GitHub source instead.

```bash
sudo add-apt-repository --remove ppa:softwareradiosystems/srsran-project
```

### Build dependencies

```bash
sudo apt-get install -y \
  cmake make gcc g++ pkg-config libfftw3-dev libmbedtls-dev \
  libsctp-dev libyaml-cpp-dev libgtest-dev libboost-program-options-dev \
  libconfig++-dev libuhd-dev uhd-host git

sudo /usr/libexec/uhd/utils/uhd_images_downloader.py
```

### Repeated `git clone` failures

```bash
cd ~
git clone https://github.com/srsran/srsRAN_Project.git
cd srsRAN_Project
mkdir build && cd build
cmake ../ -DCMAKE_BUILD_TYPE=Release -DENABLE_EXPORT=ON -DENABLE_UHD=ON
```

> ⚠️ **Issue:** the clone silently produced an incomplete repository — only `README.md` landed on disk, the rest of the ~92 MB source tree never arrived:
> ```
> CMake Error: The source directory ".../srsRAN_Project" does not appear to contain CMakeLists.txt.
> ```
> ```bash
> ls ~/srsRAN_Project/
> README.md
> ```

A stale `/usr/local/lib/libcurl.so.4` symlink was also cleaned up along the way (a noise factor, not the actual cause):

```bash
sudo rm -f /usr/local/lib/libcurl.so.4
sudo ldconfig
```

Multiple re-clones — including with verbose curl output and a shallow `--depth 1` clone — failed intermittently, pointing to an unstable network path to GitHub's codeload servers during large transfers, not a local tool misconfiguration.

✅ **Fix:** a plain retry of the standard `git clone` (no special flags) eventually succeeded once network conditions stabilized:

```bash
cd ~
rm -rf srsRAN_Project
git clone https://github.com/srsran/srsRAN_Project.git
ls ~/srsRAN_Project/
# CMakeLists.txt  apps  cmake  configs  include  lib  tests  utils  ...
```

**Fallback for unreliable connections** — a direct ZIP download as an alternative to the git protocol:

```bash
wget https://github.com/srsran/srsRAN_Project/archive/refs/heads/main.zip
```

### Successful build

```bash
cd ~/srsRAN_Project
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release -DENABLE_EXPORT=ON -DENABLE_UHD=ON
make -j$(nproc)
```

✅ **Result:** build completed, producing the `gnb` executable (srsRAN 5G gNB **v25.10.0**) plus `libsrsran_rf_uhd`, `libsrsran_rf_zmq`, and supporting libraries.

---

## Phase 5 — gNB Configuration File Debugging

This took several iterations because the YAML schema for srsRAN Project 25.10.0 differs from older examples found online — many of which use a top-level `amf:` key that belonged to an older version of the project.

### Attempt 1 — Docker-style IP addresses

The first config copied IPs (`10.53.1.1` / `10.53.1.2`) from a Docker Compose example, which didn't match the localhost-based Open5GS deployment actually running.

> ⚠️ **Issue:**
> ```
> Failed to bind UDP socket to 10.53.1.1:2152. Cannot assign requested address
> srsRAN ERROR: Unable to allocate the required NG-U network resources
> ```

✅ **Fix:** swapped the Docker-network IPs for the real loopback addresses Open5GS was bound to, confirmed via:

```bash
sudo ss -lnp | grep 38412
# Open5GS AMF listening on 127.0.0.5:38412
```

### Attempt 2 — wrong top-level key for AMF

```yaml
amf:
  addr: 127.0.0.5
  bind_addr: 127.0.0.1
```

> ⚠️ **Issue:**
> ```
> INI was not able to parse amf.++
> Run with --help for more information.
> ```

**Diagnosis:** `./gnb --help` showed AMF settings are exposed as a sub-key of `cu_cp`, not a top-level `amf:` key — confirming this build uses a CU-CP/DU-split configuration schema.

✅ **Fix:** nested the AMF block under `cu_cp:`:

```yaml
cu_cp:
  amf:
    addr: 127.0.0.5
    bind_addr: 127.0.0.1
    supported_tracking_areas:
      - tac: 7
        plmn_list:
          - plmn: "00101"
            tai_slice_support_list:
              - sst: 1
```

✅ **Result:** the AMF section parsed correctly — error moved on to a different, unrelated issue (confirming this fix was correct).

### Attempt 3 — invalid MCS / DCI format combination

> ⚠️ **Issue:**
> ```
> srsRAN ERROR: Invalid configuration DU cell detected.
> > 256QAM MCS table cannot be used for PDSCH with fallback DCI format in SearchSpace#2
> ```

✅ **Fix:** explicitly set the PDSCH and PUSCH MCS tables to `qam64` (instead of the 256QAM default), which is compatible with the fallback DCI format `1_0`/`0_0` used in SearchSpace#2:

```yaml
cell_cfg:
  pdsch:
    mcs_table: qam64
  pusch:
    mcs_table: qam64
```

### Shell heredoc corruption

While iterating on the YAML, a `cat > file << EOF ... EOF` heredoc got corrupted mid-paste in the terminal (the closing `EOF` merged with the next line), producing malformed YAML and the same `amf.++` parse error reappearing despite the content looking correct.

✅ **Fix:** switched from shell heredocs to a Python script that writes the file in one atomic operation, avoiding terminal paste/line-buffering issues entirely:

```python
python3 << 'PYEOF'
content = """cu_cp:
  amf:
    addr: 127.0.0.5
    bind_addr: 127.0.0.1
    ...
"""
with open('/home/<user>/srsRAN_Project/build/apps/gnb/gnb_srsRAN.yml', 'w') as f:
    f.write(content)
print('Done')
PYEOF
```

### Final working gNB configuration

```yaml
cu_cp:
  amf:
    addr: 127.0.0.5
    bind_addr: 127.0.0.1
    supported_tracking_areas:
      - tac: 7
        plmn_list:
          - plmn: "00101"
            tai_slice_support_list:
              - sst: 1

ru_sdr:
  device_driver: uhd
  device_args: type=b200
  clock: internal
  srate: 23.04
  tx_gain: 70
  rx_gain: 55

cell_cfg:
  dl_arfcn: 368500
  band: 3
  channel_bandwidth_MHz: 20
  common_scs: 15
  plmn: "00101"
  tac: 7
  pdcch:
    common:
      ss0_index: 0
      coreset0_index: 12
    dedicated:
      ss2_type: common
      dci_format_0_1_and_1_1: false
  prach:
    prach_config_index: 1
  pdsch:
    mcs_table: qam64
  pusch:
    mcs_table: qam64

log:
  filename: /tmp/gnb.log
  all_level: info

pcap:
  mac_enable: false
  ngap_enable: false
```

### Launching the gNB

```bash
echo "performance" | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
cd ~/srsRAN_Project/build/apps/gnb
sudo ./gnb -c gnb_srsRAN.yml
```

✅ **Result:**

```
[INFO] [B200] Detected Device: B210
[INFO] [B200] Operating over USB 3.
[INFO] [B200] Actually got clock rate 23.040000 MHz.
Cell pci=1, bw=20 MHz, 1T1R, dl_arfcn=368500 (n3), dl_freq=1842.5 MHz,
  dl_ssb_arfcn=368410, ul_freq=1747.5 MHz
N2: Connection to AMF on 127.0.0.5:38412 completed
==== gNB started ===
```

B210 detected on USB 3.0, Band n3 cell broadcasting at 20 MHz, N2/NGAP connection to the Open5GS AMF completed successfully.

---

## Phase 6 — Building and Configuring srsUE on PC1

srsRAN Project doesn't ship a UE application — the 5G NR-capable UE (`srsue`) lives in the separate `srsRAN_4G` repo, built from source on PC1.

### Build dependencies

```bash
sudo apt-get install -y \
  build-essential cmake libfftw3-dev libmbedtls-dev \
  libboost-program-options-dev libconfig++-dev libsctp-dev \
  libuhd-dev uhd-host git

# Ubuntu 24.04 requires gcc-11 specifically for this codebase
sudo apt-get install -y gcc-11 g++-11

sudo /usr/libexec/uhd/utils/uhd_images_downloader.py
```

### Building srsRAN_4G

```bash
cd ~
git clone --depth 1 https://github.com/srsran/srsRAN_4G.git
cd srsRAN_4G

export CC=$(which gcc-11)
export CXX=$(which g++-11)

mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release -DENABLE_UHD=ON
make -j$(nproc)
sudo make install
sudo ldconfig
```

✅ **Result:** build completed, producing `srsue`, `srsenb`, `srsepc`, and `srsmbms` binaries under `/usr/local/bin`.

### Config location mismatch

> ⚠️ **Issue:** `srsran_install_configs.sh` wrote default configs into `/root/.config/srsran/` instead of the regular user's home directory, because it was run via `sudo`:
> ```
> ls: cannot access '/home/<user>/.config/srsran_4g/': No such file or directory
> ```

✅ **Fix:** created the expected user config directory manually and copied the example config over:

```bash
mkdir -p ~/.config/srsran
cp /root/.config/srsran/ue.conf ~/.config/srsran/
# (also available as a packaged example:)
cp /usr/local/share/srsran/ue.conf.example ~/srsRAN_4G/build/ue.conf
```

### Building `ue.conf` for 5G NR mode

The UE config went through several rounds of correction:

- **APN** changed from a placeholder `srsapn` to `internet`, matching the Open5GS subscriber/APN config.
- **IMSI** corrected to `001010123456788` to exactly match the subscriber actually provisioned in the Open5GS WebUI.
- **`netns = ue1`** removed (set empty) — a named network namespace wasn't pre-created on the system, which would have broken TUN interface setup.
- **`device_args`** given an explicit `clock=internal` to match the gNB's own internal clock reference, since no shared GPSDO was available between the two B210s.

Final `ue.conf`:

```ini
[rf]
freq_offset = 0
tx_gain = 70
rx_gain = 60
srate = 23.04e6
nof_antennas = 1
device_name = uhd
device_args = type=b200,clock=internal
time_adv_nsamples = 300

[rat.eutra]
dl_earfcn = 2850
nof_carriers = 0

[rat.nr]
bands = 3
nof_carriers = 1
max_nof_prb = 106
nof_prb = 106

[pcap]
enable = none
mac_filename = /tmp/ue_mac.pcap
mac_nr_filename = /tmp/ue_mac_nr.pcap
nas_filename = /tmp/ue_nas.pcap

[log]
all_level = info
phy_lib_level = none
all_hex_limit = 32
filename = stdout
file_max_size = -1

[usim]
mode = soft
algo = milenage
opc  = 63BFA50EE6523365FF14C1F45F88737D
k    = 00112233445566778899aabbccddeeff
imsi = 001010123456788
imei = 353490069873319

[rrc]
release = 15
ue_category = 4

[nas]
apn = internet
apn_protocol = ipv4

[gw]
netns =
ip_devname = tun_srsue
ip_netmask = 255.255.255.0

[gui]
enable = false
```

### Launching srsUE

```bash
echo "performance" | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
sudo ~/srsRAN_4G/build/srsue/src/srsue ~/srsRAN_4G/build/ue.conf
```

---

## Phase 7 — First Successful End-to-End 5G SA Connection

With the gNB running on PC2 and srsUE launched on PC1 (B210 detected on USB 3.0 on both machines), a full connection sequence completed successfully — for the first time, end to end.

### Connection milestones

| Step | Log Evidence | Meaning |
|---|---|---|
| Cell Search | `Cell search found ARFCN=0 PCI=1 epre=-17.2 snr=+9.1` | UE detected the gNB's broadcast signal |
| SFN Sync | `SYNC: SFN synchronised successfully (SFN=350)` | Frame timing locked |
| SIB1 Acquired | `SIB1 received, CellID=4095` | System info decoded |
| Random Access | `Random Access Complete. c-rnti=0x4601, ta=4` | PRACH procedure succeeded on first attempt |
| RRC Connected | `RRC Connected` | Radio Resource Control link established |
| Authentication | `Network authentication successful` | 5G-AKA challenge/response with Open5GS AUSF/UDM passed |
| Security Mode | `Sending Security Mode Complete` | NAS encryption/integrity keys activated |
| Data Plane | `PDSCH/PUSCH mod=64QAM CRC=OK` | User-plane data successfully exchanged at 64QAM |

Raw log excerpt:

```
[RRC-NR ] Proc "Cell Selection" - Cell search found ARFCN=0 PCI=1 epre=-17.2 snr=+9.1
[PHY-SA ] SYNC: SFN synchronised successfully (SFN=350). Transitioning to IDLE...
[RRC-NR ] SIB1 received, CellID=4095
[MAC-NR ] Random Access Complete.     c-rnti=0x4601, ta=4
RRC Connected
[NAS5G  ] Network authentication successful
[NAS5G  ] Sending Security Mode Complete
[NAS5G  ] Integrity check ok. Local: count=0, Received: count=0
```

✅ **Result:** a genuine 5G Standalone over-the-air session established end-to-end — RF synchronization → System Information → Random Access → RRC → NAS Authentication → Security Mode → active data radio bearers — entirely on open-source software (srsRAN Project + Open5GS + srsRAN 4G) over two USRP B210 SDRs.

---

## Phase 8 — Clock Drift Issue on Reconnection

On a subsequent run (stopping and restarting srsUE), the UE's first Random Access attempt failed to complete contention resolution, followed by progressive loss of frame sync.

> ⚠️ **Issue:**
> ```
> [MAC-NR] [W] Contention Resolution Timer expired. Stopping PDCCH Search...
> [MAC-NR] [W] Maximum number of transmissions reached (7)
> ...
> [PHY0-NR] [E] PBCH-MIB: NR SFN (62) does not match current SFN (61)
> [PHY2-NR] [E] PBCH-MIB: NR SFN (63) does not match current SFN (62)
> [PHY0-NR] [E] PBCH-MIB: NR SFN (64) does not match current SFN (63)
> ...(gap of exactly 2 frames, growing every report)...
> [PHY-SA ] [E] SYNC: detected out-of-sync... skipping slot ...
> ```

### Diagnosis

A growing, constant 2-frame gap between the UE's tracked SFN and the gNB's broadcast SFN is the signature of **clock frequency drift** between the two USRP B210 units — not a config error. Both radios were set to `clock: internal` (each using its own free-running onboard oscillator) rather than a shared reference clock. Two independent crystal oscillators — even nominally identical ones — drift apart by a few parts-per-million over time, which is enough to desynchronize sub-frame timing within seconds at a 23.04 MHz sample rate.

This is exactly why the *first* connection of the session succeeded (clocks started close together) while a later reconnect — after the radios had been running independently for longer — failed Random Access and lost sync shortly after.

### Mitigation steps

- Reduced gNB `tx_gain` and adjusted UE `rx_gain` slightly to rule out front-end saturation as a contributing factor.
- Set explicit, matching `srate` / `master_clock_rate` (23.04 MHz) on both ends to remove any ambiguity in sample-rate negotiation.
- **Proper long-term fix (documented, not yet implemented):** drive both USRP B210 units from a shared external 10 MHz reference clock and PPS signal (e.g. a GPSDO, an Octoclock, or a simple shared reference splitter), then set `clock: external` / `sync: external` on both the gNB `ru_sdr` config and the srsUE `rf` config. This removes independent oscillator drift entirely since both radios lock to the same frequency reference.
- **Software-only partial workaround** (no shared hardware clocking available): restart both the gNB and srsUE close together in time, minimizing the free-running interval before the first PRACH attempt. Fine for short test sessions, not a real fix.

### Status

The core finding of this project — that a fully open-source 5G SA stack (srsRAN Project gNB + Open5GS core + srsRAN 4G UE) can be deployed end-to-end on commodity Ubuntu 24.04 PCs with USRP B210 radios — was conclusively demonstrated in [Phase 7](#phase-7--first-successful-end-to-end-5g-sa-connection). The clock-drift behavior here is a well-understood characteristic of free-running SDR pairs, and is solved at the hardware level (shared reference clock) for any session requiring sustained, long-duration connectivity.

---

## Appendix — Quick Command Reference

### PC2 — Service management

```bash
# Check all Open5GS core services
sudo systemctl status open5gs-amfd open5gs-upfd open5gs-nrfd \
  open5gs-smfd open5gs-ausfd open5gs-udmd open5gs-udrd

# Restart a single service after a config change
sudo systemctl restart open5gs-amfd

# Tail AMF logs live while testing the UE connection
sudo journalctl -u open5gs-amfd -f

# WebUI service
sudo systemctl restart open5gs-webui
sudo systemctl status open5gs-webui
```

### PC2 — Network / NAT setup

```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo iptables -t nat -A POSTROUTING -s 10.45.0.0/16 ! -o ogstun -j MASQUERADE
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

### Launching the stack (order matters)

1. **PC2:** confirm MongoDB is running → start Open5GS core services → start the gNB **last**.
2. **PC1:** start srsUE only after the gNB log shows `==== gNB started ===` and the N2 connection to the AMF is confirmed.

```bash
# PC2
sudo systemctl status mongod
sudo systemctl restart open5gs-amfd open5gs-smfd open5gs-upfd open5gs-nrfd open5gs-ausfd open5gs-udmd open5gs-udrd
cd ~/srsRAN_Project/build/apps/gnb && sudo ./gnb -c gnb_srsRAN.yml

# PC1 (after gNB shows 'gNB started')
sudo ~/srsRAN_4G/build/srsue/src/srsue ~/srsRAN_4G/build/ue.conf
```

### Post-connection verification

```bash
# On PC1 - check the TUN interface and connectivity
ip addr show tun_srsue
ping -I tun_srsue 8.8.8.8 -c 5

# On PC2 - confirm UE registration in AMF logs
sudo journalctl -u open5gs-amfd -n 20 --no-pager
```

---

## License

This documentation is provided under the [MIT License](LICENSE). Note that **Open5GS**, **srsRAN Project**, and **srsRAN_4G** are separate upstream projects with their own licenses — refer to each repository for details.

## Acknowledgements

- [Open5GS](https://github.com/open5gs/open5gs) — open-source 5G core network
- [srsRAN Project](https://github.com/srsran/srsRAN_Project) — open-source 5G gNB
- [srsRAN_4G](https://github.com/srsran/srsRAN_4G) — open-source 4G/5G NR UE
- [Ettus Research / UHD](https://www.ettus.com/) — USRP B210 hardware and drivers
