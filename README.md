# Open5GS 5GC & UERANSIM UE / RAN Sample Configuration - OAI-CN5G-UPF(eBPF/XDP UPF)
This describes a simple configuration for working Open5GS 5GC and OAI-CN5G-UPF(eBPF/XDP UPF).
In particular, see [here](https://github.com/s5uishida/install_oai_upf) for OAI-CN5G-UPF.

---

### [Sample Configurations and Miscellaneous for Mobile Network](https://github.com/s5uishida/sample_config_misc_for_mobile_network)

---

<a id="toc"></a>

## Table of Contents

- [Overview of Open5GS 5GC Simulation Mobile Network](#overview)
- [Changes in configuration files of Open5GS 5GC, OAI-CN5G-UPF and UERANSIM UE / RAN](#changes)
  - [Changes in configuration files of Open5GS 5GC C-Plane](#changes_cp)
  - [Changes in configuration files of OAI-CN5G-UPF](#changes_up)
  - [Changes in configuration files of UERANSIM UE / RAN](#changes_ueransim)
    - [Changes in configuration files of RAN](#changes_ran)
    - [Changes in configuration files of UE (IMSI-001010000000000)](#changes_ue)
- [Network settings of Open5GS 5GC, OAI-CN5G-UPF and UERANSIM UE / RAN](#network_settings)
  - [Network settings of Data Network Gateway](#network_settings_up)
- [Build Open5GS, OAI-CN5G-UPF and UERANSIM](#build)
- [Run Open5GS 5GC, OAI-CN5G-UPF and UERANSIM UE / RAN](#run)
  - [Run OAI-CN5G-UPF](#run_up)
  - [Run Open5GS 5GC C-Plane](#run_cp)
  - [Run UERANSIM](#run_ueran)
    - [Start gNB](#start_gnb)
    - [Start UE](#start_ue)
- [Ping google.com](#ping)
  - [Case for going through DN 10.45.0.0/16](#ping_1)
- [Changelog (summary)](#changelog)

---

<a id="overview"></a>

## Overview of Open5GS 5GC Simulation Mobile Network

This describes a simple configuration of C-Plane, eBPF/XDP UPF and Data Network Gateway for Open5GS 5GC.
**Note that this configuration is implemented with Proxmox VE VMs.**

The following minimum configuration was set as a condition.
- One UPF and Data Network Gateway
- One UE and one DNN

The built simulation environment is as follows.

<img src="./images/network-overview.png" title="./images/network-overview.png" width=1000px></img>

The 5GC / eBPF/XDP UPF / UE / RAN used are as follows.
- 5GC - Open5GS v2.8.0 (2026.09.19) - https://github.com/open5gs/open5gs
- eBPF/XDP UPF - OAI-CN5G-UPF v2.2.1 (2026.09.09) - https://github.com/openairinterface/oai-cn5g-upf
- UE / RAN - UERANSIM v3.3.0 (2026.09.06) - https://github.com/aligungr/UERANSIM

Each VMs are as follows.  
| VM | SW & Role | IP address | OS | CPU<br>(Min) | Mem<br>(Min) | HDD<br>(Min) |
| --- | --- | --- | --- | --- | --- | --- |
| VM1 | Open5GS 5GC C-Plane | 192.168.0.111/24 | Ubuntu 24.04 | 1 | 2GB | 20GB |
| VM-UP | OAI-CN5G-UPF U-Plane | 192.168.0.151/24 | Ubuntu 24.04 | 1 | 6GB | 20GB |
| VM-DN | Data Network Gateway  | 192.168.0.152/24 | Ubuntu 24.04 | 1 | 1GB | 10GB |
| VM2 | UERANSIM RAN (gNodeB) | 192.168.0.131/24 | Ubuntu 24.04 | 1 | 1GB | 10GB |
| VM3 | UERANSIM UE | 192.168.0.132/24 | Ubuntu 24.04 | 1 | 1GB | 10GB |

The network interfaces of each VM are as follows.
| VM | Device | Model | Linux Bridge | IP address | Interface | XDP |
| --- | --- | --- | --- | --- | --- | --- |
| VM1 | ens18 | VirtIO | vmbr1 | 10.0.0.111/24 | (NAPT NW) | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.111/24 | (Mgmt NW) | -- |
| | ens20 | VirtIO | vmbr4 | 192.168.14.111/24 | N4 | -- |
| VM-UP | ~~ens18~~ | ~~VirtIO~~ | ~~vmbr1~~ | ~~10.0.0.151/24~~ | ~~(NAPT NW)~~ ***down*** | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.151/24 | (Mgmt NW) | -- |
| | ens20 | VirtIO | vmbr3 | 192.168.13.151/24 | N3 | x |
| | ens21 | VirtIO | vmbr4 | 192.168.14.151/24 | N4 | -- |
| | ens22 | VirtIO | vmbr6 | 192.168.16.151/24 | N6 | x |
| VM-DN | ens18 | VirtIO | vmbr1 | 10.0.0.152/24 | (NAPT NW) | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.152/24 | (Mgmt NW) | -- |
| | ens20 | VirtIO | vmbr6 | 192.168.16.152/24 | N6 | -- |
| VM2 | ens18 | VirtIO | vmbr1 | 10.0.0.131/24 | (NAPT NW) | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.131/24 | (Mgmt NW) | -- |
| | ens20 | VirtIO | vmbr3 | 192.168.13.131/24 | N3 | -- |
| VM3 | ens18 | VirtIO | vmbr1 | 10.0.0.132/24 | (NAPT NW) | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.132/24 | (Mgmt NW) | -- |

Linux Bridges of Proxmox VE are as follows.
| Linux Bridge | Network CIDR | Interface |
| --- | --- | --- |
| vmbr1 | 10.0.0.0/24 | NAPT NW |
| mgbr0 | 192.168.0.0/24 | Mgmt NW |
| vmbr3 | 192.168.13.0/24 | N3 |
| vmbr4 | 192.168.14.0/24 | N4 |
| vmbr6 | 192.168.16.0/24 | N6 |

Subscriber Information (other information is the same) is as follows.  
**Note. Please select OP or OPc according to the setting of UERANSIM UE configuration file.**
| UE | IMSI | DNN | OP/OPc |
| --- | --- | --- | --- |
| UE | 001010000000000 | internet | OPc |

I registered these information with the Open5GS WebUI.
In addition, [3GPP TS 35.208](https://www.3gpp.org/DynaReport/35208.htm) "4.3 Test Sets" is published by 3GPP as test data for the 3GPP authentication and key generation functions (MILENAGE).

The DN is as follows.
| DN | DNN | TUNnel interface of UE |
| --- | --- | --- |
| 10.45.0.0/16 | internet | uesimtun0 |

<a id="changes"></a>

## Changes in configuration files of Open5GS 5GC, OAI-CN5G-UPF and UERANSIM UE / RAN

Please refer to the following for building Open5GS, OAI-CN5G-UPF and UERANSIM respectively.
- Open5GS v2.8.0 (2026.09.19) - https://open5gs.org/open5gs/docs/guide/02-building-open5gs-from-sources/
- OAI-CN5G-UPF v2.2.1 (2026.09.09) - https://github.com/s5uishida/install_oai_upf
- UERANSIM v3.3.0 (2026.09.06) - https://github.com/aligungr/UERANSIM/wiki/Installation

<a id="changes_cp"></a>

### Changes in configuration files of Open5GS 5GC C-Plane

The following parameters can be used in the logic that selects UPF as the connection destination by PFCP.

- DNN
- TAC (Tracking Area Code)
- nr_CellID

For the sake of simplicity, I used only DNN this time.

- `open5gs/install/etc/open5gs/amf.yaml`
```diff
--- amf.yaml.orig       2026-09-16 20:29:38.000000000 +0900
+++ amf.yaml    2026-09-20 00:00:38.776922880 +0900
@@ -20,27 +20,27 @@
         - uri: http://127.0.0.200:7777
   ngap:
     server:
-      - address: 127.0.0.5
+      - address: 192.168.0.111
   metrics:
     server:
       - address: 127.0.0.5
         port: 9090
   guami:
     - plmn_id:
-        mcc: 999
-        mnc: 70
+        mcc: 001
+        mnc: 01
       amf_id:
         region: 2
         set: 1
   tai:
     - plmn_id:
-        mcc: 999
-        mnc: 70
+        mcc: 001
+        mnc: 01
       tac: 1
   plmn_support:
     - plmn_id:
-        mcc: 999
-        mnc: 70
+        mcc: 001
+        mnc: 01
       s_nssai:
         - sst: 1
   security:
```
- `open5gs/install/etc/open5gs/nrf.yaml`
```diff
--- nrf.yaml.orig       2025-01-15 04:12:06.000000000 +0900
+++ nrf.yaml    2025-01-15 04:22:49.000000000 +0900
@@ -11,8 +11,8 @@
 nrf:
   serving:  # 5G roaming requires PLMN in NRF
     - plmn_id:
-        mcc: 999
-        mnc: 70
+        mcc: 001
+        mnc: 01
   sbi:
     server:
       - address: 127.0.0.10
```
- `open5gs/install/etc/open5gs/smf.yaml`
```diff
--- smf.yaml.orig       2025-01-15 04:12:06.000000000 +0900
+++ smf.yaml    2025-01-15 04:26:36.000000000 +0900
@@ -20,16 +20,14 @@
         - uri: http://127.0.0.200:7777
   pfcp:
     server:
-      - address: 127.0.0.4
+      - address: 192.168.14.111
     client:
       upf:
-        - address: 127.0.0.7
-  gtpc:
-    server:
-      - address: 127.0.0.4
+        - address: 192.168.14.151
+          dnn: internet
   gtpu:
     server:
-      - address: 127.0.0.4
+      - address: 192.168.14.111
   metrics:
     server:
       - address: 127.0.0.4
@@ -37,20 +35,17 @@
   session:
     - subnet: 10.45.0.0/16
       gateway: 10.45.0.1
-    - subnet: 2001:db8:cafe::/48
-      gateway: 2001:db8:cafe::1
+      dnn: internet
   dns:
     - 8.8.8.8
     - 8.8.4.4
-    - 2001:4860:4860::8888
-    - 2001:4860:4860::8844
   mtu: 1400
 #  p-cscf:
 #    - 127.0.0.1
 #    - ::1
 #  ctf:
 #    enabled: auto   # auto(default)|yes|no
-  freeDiameter: /root/open5gs/install/etc/freeDiameter/smf.conf
+#  freeDiameter: /root/open5gs/install/etc/freeDiameter/smf.conf
 
 ################################################################################
 # SMF Info
```

<a id="changes_up"></a>

### Changes in configuration files of OAI-CN5G-UPF

See [here](https://github.com/s5uishida/install_oai_upf#conf) for the original file.

- `openair-upf/config.yaml`  
There is no change.

<a id="changes_ueransim"></a>

### Changes in configuration files of UERANSIM UE / RAN

<a id="changes_ran"></a>

#### Changes in configuration files of RAN

- `UERANSIM/config/open5gs-gnb.yaml`
```diff
--- open5gs-gnb.yaml.orig       2026-09-07 02:39:26.000000000 +0900
+++ open5gs-gnb.yaml    2026-09-20 01:07:43.623080002 +0900
@@ -1,17 +1,17 @@
-mcc: '999'          # Mobile Country Code value
-mnc: '70'           # Mobile Network Code value (2 or 3 digits)
+mcc: '001'          # Mobile Country Code value
+mnc: '01'           # Mobile Network Code value (2 or 3 digits)
 
 nci: '0x000000010'  # NR Cell Identity (36-bit)
 idLength: 32        # NR gNB ID length in bits [22...32]
 tac: 1              # Tracking Area Code
 
-linkIp: 127.0.0.1   # gNB's local IP address for Radio Link Simulation (Usually same with local IP)
-ngapIp: 127.0.0.1   # gNB's local IP address for N2 Interface (Usually same with local IP)
-gtpIp: 127.0.0.1    # gNB's local IP address for N3 Interface (Usually same with local IP)
+linkIp: 192.168.0.131   # gNB's local IP address for Radio Link Simulation (Usually same with local IP)
+ngapIp: 192.168.0.131   # gNB's local IP address for N2 Interface (Usually same with local IP)
+gtpIp: 192.168.13.131    # gNB's local IP address for N3 Interface (Usually same with local IP)
 
 # List of AMF address information
 amfConfigs:
-  - address: 127.0.0.5
+  - address: 192.168.0.111
     port: 38412
 
 # List of supported S-NSSAIs by this gNB
```

<a id="changes_ue"></a>

#### Changes in configuration files of UE (IMSI-001010000000000)

- `UERANSIM/config/open5gs-ue.yaml`
```diff
--- open5gs-ue.yaml.orig        2026-09-07 04:20:36.000000000 +0900
+++ open5gs-ue.yaml     2026-09-20 01:11:26.101597837 +0900
@@ -1,9 +1,9 @@
 # IMSI number of the UE. IMSI = [MCC|MNC|MSISDN] (In total 15 digits)
-supi: 'imsi-999700000000001'
+supi: 'imsi-001010000000000'
 # Mobile Country Code value of HPLMN
-mcc: '999'
+mcc: '001'
 # Mobile Network Code value of HPLMN (2 or 3 digits)
-mnc: '70'
+mnc: '01'
 # SUCI Protection Scheme : 0 for Null-scheme, 1 for Profile A and 2 for Profile B
 protectionScheme: 0
 # Home Network Public Key for protecting with SUCI
@@ -39,7 +39,7 @@
 
 # List of gNB IP addresses for Radio Link Simulation
 gnbSearchList:
-  - 127.0.0.1
+  - 192.168.0.131
 
 # UAC Access Identities Configuration
 uacAic:
```

<a id="network_settings"></a>

## Network settings of Open5GS 5GC, OAI-CN5G-UPF and UERANSIM UE / RAN

<a id="network_settings_up"></a>

### Network settings of Data Network Gateway

See [this](https://github.com/s5uishida/install_oai_upf#setup_dn).

<a id="build"></a>

## Build Open5GS, OAI-CN5G-UPF and UERANSIM

Please refer to the following for building Open5GS, OAI-CN5G-UPF and UERANSIM respectively.
- Open5GS v2.8.0 (2026.09.19) - https://open5gs.org/open5gs/docs/guide/02-building-open5gs-from-sources/
- OAI-CN5G-UPF v2.2.1 (2026.09.09) - https://github.com/s5uishida/install_oai_upf
- UERANSIM v3.3.0 (2026.09.06) - https://github.com/aligungr/UERANSIM/wiki/Installation

Install MongoDB on Open5GS 5GC C-Plane machine.
[MongoDB Compass](https://www.mongodb.com/products/compass) is a convenient tool to look at the MongoDB database.

<a id="run"></a>

## Run Open5GS 5GC, OAI-CN5G-UPF and UERANSIM UE / RAN

First run OAI-CN5G-UPF, then the 5GC and UERANSIM (UE & RAN implementation).

<a id="run_up"></a>

### Run OAI-CN5G-UPF

See [this](https://github.com/s5uishida/install_oai_upf#run).

<a id="run_cp"></a>

### Run Open5GS 5GC C-Plane

```
./install/bin/open5gs-nrfd &
sleep 2
./install/bin/open5gs-scpd &
sleep 2
./install/bin/open5gs-amfd &
sleep 2
./install/bin/open5gs-smfd &
./install/bin/open5gs-ausfd &
./install/bin/open5gs-udmd &
./install/bin/open5gs-udrd &
./install/bin/open5gs-pcfd &
./install/bin/open5gs-nssfd &
./install/bin/open5gs-bsfd &
./install/bin/open5gs-eird &
```
The PFCP association log between OAI-CN5G-UPF and Open5GS SMF is as follows.
```
[2026-09-20 01:21:09.634] [upf_n4 ] [info] handle_receive(30 bytes)
[2026-09-20 01:21:09.634] [upf_n4 ] [info] Handle SX ASSOCIATION SETUP REQUEST
[2026-09-20 01:21:09.635] [upf_n4 ] [info] handle_receive(16 bytes)
[2026-09-20 01:21:09.635] [upf_n4 ] [info] Received SX HEARTBEAT REQUEST
```

<a id="run_ueran"></a>

### Run UERANSIM

Here, the case of UE (IMSI-001010000000000) & RAN is described.
First, do an NG Setup between gNodeB and 5GC, then register the UE with 5GC and establish a PDU session.

Please refer to the following for usage of UERANSIM.

https://github.com/aligungr/UERANSIM/wiki/Usage

<a id="start_gnb"></a>

#### Start gNB

Start gNB as follows.
```
# ./nr-gnb -c ../config/open5gs-gnb.yaml
UERANSIM v3.3.0
[2026-09-20 01:21:30.938] [sctp] [info] Trying to establish SCTP connection... (192.168.0.111:38412)
[2026-09-20 01:21:30.976] [sctp] [info] SCTP connection established (192.168.0.111:38412)
[2026-09-20 01:21:30.976] [sctp] [debug] SCTP association setup ascId[3]
[2026-09-20 01:21:30.976] [ngap] [debug] Sending NG Setup Request
[2026-09-20 01:21:30.982] [ngap] [debug] NG Setup Response received
[2026-09-20 01:21:30.982] [ngap] [info] NG Setup procedure is successful
```
The Open5GS C-Plane log when executed is as follows.
```
09/20 01:21:31.687: [amf] INFO: gNB-N2 accepted[192.168.0.131]:58432 in ng-path module (../src/amf/ngap-sctp.c:113)
09/20 01:21:31.687: [amf] INFO: gNB-N2 accepted[192.168.0.131] in master_sm module (../src/amf/amf-sm.c:823)
09/20 01:21:31.693: [amf] INFO: [Added] Number of gNBs is now 1 (../src/amf/context.c:1349)
09/20 01:21:31.693: [amf] INFO: gNB-N2[192.168.0.131] max_num_of_ostreams : 10 (../src/amf/amf-sm.c:870)
```

<a id="start_ue"></a>

#### Start UE

Start UE as follows. This will register the UE with 5GC and establish a PDU session.
```
# ./nr-ue -c ../config/open5gs-ue.yaml
UERANSIM v3.3.0
[2026-09-20 01:21:41.877] [nas] [info] UE switches to state [MM-DEREGISTERED/PLMN-SEARCH]
[2026-09-20 01:21:41.878] [rrc] [debug] New signal detected for cell[1], total [1] cells in coverage
[2026-09-20 01:21:41.878] [nas] [info] Selected plmn[001/01]
[2026-09-20 01:21:41.878] [rrc] [info] Selected cell plmn[001/01] tac[1] category[SUITABLE]
[2026-09-20 01:21:41.878] [nas] [info] UE switches to state [MM-DEREGISTERED/PS]
[2026-09-20 01:21:41.878] [nas] [info] UE switches to state [MM-DEREGISTERED/NORMAL-SERVICE]
[2026-09-20 01:21:41.879] [nas] [debug] Initial registration required due to [MM-DEREG-NORMAL-SERVICE]
[2026-09-20 01:21:41.879] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2026-09-20 01:21:41.879] [nas] [debug] Sending Initial Registration
[2026-09-20 01:21:41.880] [rrc] [debug] Sending RRC Setup Request
[2026-09-20 01:21:41.880] [nas] [info] UE switches to state [MM-REGISTER-INITIATED]
[2026-09-20 01:21:41.880] [rrc] [info] RRC connection established
[2026-09-20 01:21:41.880] [rrc] [info] UE switches to state [RRC-CONNECTED]
[2026-09-20 01:21:41.880] [nas] [info] UE switches to state [CM-CONNECTED]
[2026-09-20 01:21:41.886] [nas] [debug] Authentication Request received
[2026-09-20 01:21:41.886] [nas] [debug] Received SQN [0000000011E1]
[2026-09-20 01:21:41.886] [nas] [debug] SQN-MS [000000000000]
[2026-09-20 01:21:41.891] [nas] [debug] Security Mode Command received
[2026-09-20 01:21:41.891] [nas] [debug] Selected integrity[2] ciphering[0]
[2026-09-20 01:21:41.902] [nas] [debug] Registration accept received
[2026-09-20 01:21:41.902] [nas] [info] UE switches to state [MM-REGISTERED/NORMAL-SERVICE]
[2026-09-20 01:21:41.902] [nas] [debug] Sending Registration Complete
[2026-09-20 01:21:41.902] [nas] [info] Initial Registration is successful
[2026-09-20 01:21:41.902] [nas] [debug] Sending PDU Session Establishment Request
[2026-09-20 01:21:41.902] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2026-09-20 01:21:42.106] [nas] [debug] Configuration Update Command received
[2026-09-20 01:21:42.121] [nas] [debug] PDU Session Establishment Accept received
[2026-09-20 01:21:42.122] [nas] [info] PDU Session establishment is successful PSI[1]
[2026-09-20 01:21:42.153] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.45.0.2] is up.
```
The Open5GS C-Plane log when executed is as follows.
```
09/20 01:21:42.566: [amf] INFO: InitialUEMessage (../src/amf/ngap-handler.c:668)
09/20 01:21:42.566: [amf] INFO: [Added] Number of gNB-UEs is now 1 (../src/amf/context.c:3049)
09/20 01:21:42.566: [amf] INFO:     RAN_UE_NGAP_ID[1] AMF_UE_NGAP_ID[1] TAC[1] CellID[0x10] (../src/amf/ngap-handler.c:884)
09/20 01:21:42.566: [amf] INFO: [suci-0-001-01-0000-0-0-0000000000] Unknown UE by SUCI (../src/amf/context.c:2064)
09/20 01:21:42.566: [amf] INFO: [Added] Number of AMF-UEs is now 1 (../src/amf/context.c:1818)
09/20 01:21:42.566: [gmm] INFO: Registration request (../src/amf/gmm-sm.c:1709)
09/20 01:21:42.566: [gmm] INFO: [suci-0-001-01-0000-0-0-0000000000]    SUCI (../src/amf/gmm-handler.c:186)
09/20 01:21:42.566: [sbi] INFO: [200f4f7a-b446-41f1-b207-f5cd673a1d22] Setup NF Instance [type:AUSF] (../lib/sbi/path.c:349)
09/20 01:21:42.566: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.11:7777] (../src/scp/sbi-path.c:583)
09/20 01:21:42.567: [sbi] INFO: [200f9eda-b446-41f1-a975-713325b1a347] Setup NF Instance [type:UDM] (../lib/sbi/path.c:349)
09/20 01:21:42.567: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.12:7777] (../src/scp/sbi-path.c:583)
09/20 01:21:42.567: [sbi] INFO: [2011c8e0-b446-41f1-8b76-a3e800e4957c] Setup NF Instance [type:UDR] (../lib/sbi/path.c:349)
09/20 01:21:42.568: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.20:7777] (../src/scp/sbi-path.c:583)
09/20 01:21:42.570: [amf] INFO: Setup NF EndPoint(addr) [127.0.0.11:7777] (../src/amf/nausf-handler.c:152)
09/20 01:21:42.572: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.11:7777] (../src/scp/sbi-path.c:583)
09/20 01:21:42.572: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.12:7777] (../src/scp/sbi-path.c:583)
09/20 01:21:42.573: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.20:7777] (../src/scp/sbi-path.c:583)
09/20 01:21:42.575: [ausf] INFO: Setup NF EndPoint(addr) [127.0.0.12:7777] (../src/ausf/nudm-handler.c:339)
09/20 01:21:42.577: [gmm] INFO: [imsi-001010000000000] Security mode complete (../src/amf/gmm-sm.c:2784)
09/20 01:21:42.577: [gmm] INFO: [imsi-001010000000000] Skip 5G-EIR check [message:65,enabled:0] (../src/amf/gmm-sm.c:2683)
09/20 01:21:42.577: [sbi] INFO: [200f9eda-b446-41f1-a975-713325b1a347] Setup NF Instance [type:UDM] (../lib/sbi/path.c:349)
09/20 01:21:42.577: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.12:7777] (../src/scp/sbi-path.c:583)
09/20 01:21:42.578: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.20:7777] (../src/scp/sbi-path.c:583)
09/20 01:21:42.579: [sbi] INFO: [200f9eda-b446-41f1-a975-713325b1a347] Setup NF Instance [type:UDM] (../lib/sbi/path.c:349)
09/20 01:21:42.579: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.12:7777] (../src/scp/sbi-path.c:583)
09/20 01:21:42.580: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.20:7777] (../src/scp/sbi-path.c:583)
09/20 01:21:42.581: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.12:7777] (../src/scp/sbi-path.c:583)
09/20 01:21:42.582: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.12:7777] (../src/scp/sbi-path.c:583)
09/20 01:21:42.583: [amf] INFO: Setup NF EndPoint(addr) [127.0.0.12:7777] (../src/amf/nudm-handler.c:431)
09/20 01:21:42.583: [sbi] INFO: [2011fe0a-b446-41f1-b166-f3395f4fe667] Setup NF Instance [type:PCF] (../lib/sbi/path.c:349)
09/20 01:21:42.583: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.13:7777] (../src/scp/sbi-path.c:583)
09/20 01:21:42.584: [pcf] INFO: Setup NF EndPoint(addr) [127.0.0.5:7777] (../src/pcf/npcf-handler.c:210)
09/20 01:21:42.584: [sbi] INFO: [2011c8e0-b446-41f1-8b76-a3e800e4957c] Setup NF Instance [type:UDR] (../lib/sbi/path.c:349)
09/20 01:21:42.584: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.20:7777] (../src/scp/sbi-path.c:583)
09/20 01:21:42.586: [amf] INFO: Setup NF EndPoint(addr) [127.0.0.13:7777] (../src/amf/npcf-handler.c:143)
09/20 01:21:42.790: [gmm] INFO: [imsi-001010000000000] Registration complete (../src/amf/gmm-sm.c:3458)
09/20 01:21:42.790: [amf] INFO: [imsi-001010000000000] Configuration update command (../src/amf/nas-path.c:609)
09/20 01:21:42.790: [gmm] INFO:     UTC [2026-09-19T16:21:42] Timezone[0]/DST[0] (../src/amf/gmm-build.c:556)
09/20 01:21:42.790: [gmm] INFO:     LOCAL [2026-09-20T01:21:42] Timezone[32400]/DST[0] (../src/amf/gmm-build.c:561)
09/20 01:21:42.790: [amf] INFO: [Added] Number of AMF-Sessions is now 1 (../src/amf/context.c:3070)
09/20 01:21:42.790: [gmm] INFO: UE SUPI[imsi-001010000000000] DNN[internet] LBO[0] S_NSSAI[SST:1 SD:0xffffff] smContextRef[NULL] smContextResourceURI[NULL] (../src/amf/gmm-handler.c:1452)
09/20 01:21:42.790: [gmm] INFO: V-SMF Instance [2025a6b2-b446-41f1-b2a7-a934c4475258](LIST) (../src/amf/gmm-handler.c:1529)
09/20 01:21:42.790: [gmm] INFO: [2025a6b2-b446-41f1-b2a7-a934c4475258] Setup NF Instance [type:SMF] (../src/amf/gmm-handler.c:1531)
09/20 01:21:42.790: [gmm] INFO: V-SMF Instance [2025a6b2-b446-41f1-b2a7-a934c4475258] (../src/amf/gmm-handler.c:1541)
09/20 01:21:42.790: [gmm] INFO: V-SMF discovered in Non-Roaming or LBO-Roaming[0] (../src/amf/gmm-handler.c:1610)
09/20 01:21:42.790: [gmm] INFO: nsmf_pdusession [1:0x61b4be0f4ef8:(nil)] (../src/amf/gmm-handler.c:1650)
09/20 01:21:42.791: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.4:7777] (../src/scp/sbi-path.c:583)
09/20 01:21:42.791: [smf] INFO: [Added] Number of SMF-UEs is now 1 (../src/smf/context.c:1069)
09/20 01:21:42.791: [smf] INFO: [Added] Number of SMF-Sessions is now 1 (../src/smf/context.c:3637)
09/20 01:21:42.791: [smf] INFO: Setup NF EndPoint(addr) [127.0.0.5:7777] (../src/smf/nsmf-handler.c:326)
09/20 01:21:42.792: [sbi] INFO: [200f9eda-b446-41f1-a975-713325b1a347] Setup NF Instance [type:UDM] (../lib/sbi/path.c:349)
09/20 01:21:42.792: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.12:7777] (../src/scp/sbi-path.c:583)
09/20 01:21:42.792: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.20:7777] (../src/scp/sbi-path.c:583)
09/20 01:21:42.794: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.12:7777] (../src/scp/sbi-path.c:583)
09/20 01:21:42.795: [smf] INFO: Setup NF EndPoint(addr) [127.0.0.12:7777] (../src/smf/nudm-handler.c:473)
09/20 01:21:42.795: [sbi] INFO: [2011fe0a-b446-41f1-b166-f3395f4fe667] Setup NF Instance [type:PCF] (../lib/sbi/path.c:349)
09/20 01:21:42.796: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.13:7777] (../src/scp/sbi-path.c:583)
09/20 01:21:42.796: [amf] INFO: Setup NF EndPoint(addr) [127.0.0.4:7777] (../src/amf/nsmf-handler.c:140)
09/20 01:21:42.797: [pcf] INFO: Setup NF EndPoint(addr) [127.0.0.4:7777] (../src/pcf/npcf-handler.c:542)
09/20 01:21:42.797: [sbi] INFO: [2011c8e0-b446-41f1-8b76-a3e800e4957c] Setup NF Instance [type:UDR] (../lib/sbi/path.c:349)
09/20 01:21:42.797: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.20:7777] (../src/scp/sbi-path.c:583)
09/20 01:21:42.798: [sbi] INFO: [200ff2fe-b446-41f1-945d-4754696b0104] Setup NF Instance [type:BSF] (../lib/sbi/path.c:349)
09/20 01:21:42.798: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.15:7777] (../src/scp/sbi-path.c:583)
09/20 01:21:42.799: [pcf] INFO: Setup NF EndPoint(addr) [127.0.0.15:7777] (../src/pcf/nbsf-handler.c:125)
09/20 01:21:42.800: [smf] INFO: Setup NF EndPoint(addr) [127.0.0.13:7777] (../src/smf/npcf-handler.c:414)
09/20 01:21:42.800: [smf] INFO: UE SUPI[imsi-001010000000000] DNN[internet] IPv4[10.45.0.2] IPv6[] (../src/smf/npcf-handler.c:657)
09/20 01:21:42.802: [gtp] INFO: gtp_connect() [192.168.13.151]:2152 (../lib/gtp/path.c:60)
09/20 01:21:42.803: [sbi] INFO: [1ed64334-b446-41f1-bb9c-79789c347dd8] Setup NF Instance [type:AMF] (../lib/sbi/path.c:349)
09/20 01:21:42.803: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.5:7777] (../src/scp/sbi-path.c:583)
09/20 01:21:42.806: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.4:7777] (../src/scp/sbi-path.c:583)
09/20 01:21:42.844: [sbi] INFO: [200f9eda-b446-41f1-a975-713325b1a347] Setup NF Instance [type:UDM] (../lib/sbi/path.c:349)
09/20 01:21:42.844: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.12:7777] (../src/scp/sbi-path.c:583)
09/20 01:21:42.844: [sbi] INFO: [2011c8e0-b446-41f1-8b76-a3e800e4957c] Setup NF Instance [type:UDR] (../lib/sbi/path.c:349)
09/20 01:21:42.845: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.20:7777] (../src/scp/sbi-path.c:583)
09/20 01:21:42.846: [amf] INFO: [imsi-001010000000000:1:11][0:0:NULL] /nsmf-pdusession/v1/sm-contexts/{smContextRef}/modify (../src/amf/nsmf-handler.c:1036)
```
The PDU session establishment log of OAI-CN5G-UPF is as follows.
```
[2026-09-20 01:21:41.903] [upf_n4 ] [info] handle_receive(596 bytes)
[2026-09-20 01:21:41.903] [upf_app] [info] 
[2026-09-20 01:21:41.903] [upf_app] [info] ╔═════════════════════════════════════════════════════════════════════════════╗
[2026-09-20 01:21:41.903] [upf_app] [info] │             Received N4_SESSION_ESTABLISHMENT_REQUEST seid 0x0              │
[2026-09-20 01:21:41.903] [upf_app] [info] ╚═════════════════════════════════════════════════════════════════════════════╝
[2026-09-20 01:21:41.904] [upf_n4 ] [info] pfcp_session::add(far) seid 0x1 FAR=1
[2026-09-20 01:21:41.904] [upf_n4 ] [info]   └─ Adding new FAR 1 to session 0x1
[2026-09-20 01:21:41.904] [upf_n4 ] [info] pfcp_session::add(far) seid 0x1 FAR=2
[2026-09-20 01:21:41.904] [upf_n4 ] [info]   └─ Adding new FAR 2 to session 0x1
[2026-09-20 01:21:41.904] [upf_n4 ] [info] pfcp_session::add(far) seid 0x1 FAR=3
[2026-09-20 01:21:41.904] [upf_n4 ] [info]   └─ Adding new FAR 3 to session 0x1
[2026-09-20 01:21:41.904] [upf_n4 ] [info] pfcp_session::add(qer) seid 0x1 QER=1
[2026-09-20 01:21:41.904] [upf_n4 ] [info]   └─ Adding new QER 1 to session 0x1
[2026-09-20 01:21:41.904] [upf_n4 ] [info] pfcp_session::add(pdr) seid 0x1 PDR=1
[2026-09-20 01:21:41.904] [upf_n4 ] [info]   └─ Adding new PDR 1 to session 0x1
[2026-09-20 01:21:41.904] [upf_n4 ] [info] pfcp_session::add(qer) seid 0x1 QER=1
[2026-09-20 01:21:41.904] [upf_n4 ] [warning]   └─ Skipping duplicate QER 1 (QFI 1) in session 0x1 - already exists
[2026-09-20 01:21:41.904] [upf_n4 ] [info] pfcp_session::set(fteid) seid 0x1 
[2026-09-20 01:21:41.904] [upf_n4 ] [info] pfcp_session::get(fteid) seid 0x1 
[2026-09-20 01:21:41.904] [upf_n4 ] [info] pfcp_session::add(pdr) seid 0x1 PDR=2
[2026-09-20 01:21:41.904] [upf_n4 ] [info]   └─ Adding new PDR 2 to session 0x1
[2026-09-20 01:21:41.904] [upf_n4 ] [info] pfcp_session::set(fteid) seid 0x1 
[2026-09-20 01:21:41.904] [upf_n4 ] [info] pfcp_session::get(fteid) seid 0x1 
[2026-09-20 01:21:41.904] [upf_n4 ] [info] pfcp_session::add(pdr) seid 0x1 PDR=3
[2026-09-20 01:21:41.904] [upf_n4 ] [info]   └─ Adding new PDR 3 to session 0x1
[2026-09-20 01:21:41.904] [upf_n4 ] [info] pfcp_session::set(fteid) seid 0x1 
[2026-09-20 01:21:41.904] [upf_n4 ] [info] pfcp_session::get(fteid) seid 0x1 
[2026-09-20 01:21:41.904] [upf_n4 ] [info] pfcp_session::add(pdr) seid 0x1 PDR=4
[2026-09-20 01:21:41.904] [upf_n4 ] [info]   └─ Adding new PDR 4 to session 0x1
[2026-09-20 01:21:41.904] [upf_app] [info] Establish datapath: create(pdr(s), far(s), qer(s), urr(s), bar(s), mar(s))
[2026-09-20 01:21:41.904] [upf_n4 ] [info] Unhandled source interface for PDR: 3
[2026-09-20 01:21:41.904] [upf_app] [info] [eBPF] Create Pipeline - Creating pipeline for session 0x1
[2026-09-20 01:21:41.904] [upf_app] [warning] F-TEID missing for PDR 1 (CH bit: Not Set)
[2026-09-20 01:21:41.904] [upf_app] [warning] UE IP address missing for PDR 2
[2026-09-20 01:21:41.904] [upf_app] [warning] UE IP address missing for PDR 3
[2026-09-20 01:21:41.904] [upf_app] [warning] UE IP address missing for PDR 4
[2026-09-20 01:21:41.904] [upf_app] [info] Pipeline created for session 0x1 with 4 PDRs [type = IP, rules = 0x1]
[2026-09-20 01:21:41.904] [upf_app] [info] [N4] Create Session: seid 0x1 - eBPF data-path pipeline created successfully
[2026-09-20 01:21:41.910] [upf_n4 ] [info] handle_receive(75 bytes)
[2026-09-20 01:21:41.911] [upf_app] [info] 
[2026-09-20 01:21:41.911] [upf_app] [info] ╔═════════════════════════════════════════════════════════════════════════════╗
[2026-09-20 01:21:41.911] [upf_app] [info] │             Received N4_SESSION_MODIFICATION_REQUEST seid 0x1               │
[2026-09-20 01:21:41.911] [upf_app] [info] ╚═════════════════════════════════════════════════════════════════════════════╝
[2026-09-20 01:21:41.911] [upf_n4 ] [info] pfcp_session::update(far) seid 0x1 FAR=1
[2026-09-20 01:21:41.911] [upf_n4 ] [info]   └─ Updating FAR 1 in session 0x1
[2026-09-20 01:21:41.911] [upf_app] [info] Modify datapath
[2026-09-20 01:21:41.911] [upf_app] [info] [eBPF] Modify Pipeline - Updating pipeline for session 0x1
[2026-09-20 01:21:41.911] [upf_app] [warning] Session 0x1 has no TEIDs - PDU session mapping not updated
[2026-09-20 01:21:41.911] [upf_app] [info] [eBPF] Modify Pipeline - Pipeline modified for session 0x1 with 4 PDRs (0 uplink TEIDs, 0 downlink TEIDs)
[2026-09-20 01:21:41.911] [upf_app] [info] [N4] Session Modification: seid 0x1
[2026-09-20 01:21:41.911] [upf_app] [info]   └─ Updated: 0 PDR, 1 FAR, 0 QER, 0 URR, 0 BAR, 0 MAR
[2026-09-20 01:21:41.911] [upf_n4 ] [info] Unhandled source interface for PDR: 3
[2026-09-20 01:21:41.911] [upf_app] [info] [eBPF] Modify Pipeline - Updating pipeline for session 0x1
[2026-09-20 01:21:41.912] [upf_app] [info] 
[2026-09-20 01:21:41.912] [upf_app] [info]   ┌───────────────────────────────────────────────────┐
[2026-09-20 01:21:41.912] [upf_app] [info]   │            QoS ENFORCEMENT SETUP                  │
[2026-09-20 01:21:41.912] [upf_app] [info]   │       Session: 0x1, Interface: ens20              │
[2026-09-20 01:21:41.912] [upf_app] [info]   └───────────────────────────────────────────────────┘
[2026-09-20 01:21:41.912] [upf_app] [info]   ┌─ N6 Interface (Non-GTP): ens22
[2026-09-20 01:21:41.912] [upf_app] [info]   └─ N3 Interface (GTP):     ens20
[2026-09-20 01:21:41.931] [upf_app] [info]   ┌─ Creating Root HTB Qdisc on ens20
[2026-09-20 01:21:41.931] [upf_app] [info]   │  • Default Class: 65535
[2026-09-20 01:21:41.931] [upf_app] [info]   │  • r2q Parameter: 1000
[2026-09-20 01:21:41.934] [upf_app] [info]   └─ ✓ Root qdisc created successfully on interface: ens20
[2026-09-20 01:21:41.934] [upf_app] [info]   ┌─ Creating PDU Session Class 1:1
[2026-09-20 01:21:41.934] [upf_app] [info]   │  • Session Rate: 4,294,966,296 kbps
[2026-09-20 01:21:41.936] [upf_app] [info]   └─ ✓ PDU session class  1:1 created successfully
[2026-09-20 01:21:41.936] [upf_app] [warning] QoS Flow missing GBR: set it to 0.8 x MBR
[2026-09-20 01:21:41.936] [upf_app] [info]   ┌─ Createing QoS Flow Class 1:38 for PDU Session Parent 1:1
[2026-09-20 01:21:41.936] [upf_app] [info]   │  • QoS Flow Rate (GBR): 160,000,000 kbps
[2026-09-20 01:21:41.936] [upf_app] [info]   │  • QoS Flow Ceil (MBR): 200,000,000 kbps
Warning: sch_htb: quantum of class 10026 is big. Consider r2q change.
[2026-09-20 01:21:41.938] [upf_app] [info]   └─ ✓ QoS Flow class  1:38 created successfully for QER 1
[2026-09-20 01:21:41.942] [upf_app] [info] Attach Section tc_filter_traffic to gtp interface
[2026-09-20 01:21:41.946] [upf_app] [info] Attach Section tc_redirect to udp interface
libbpf: Kernel error message: Exclusivity flag on, cannot modify
[2026-09-20 01:21:41.946] [upf_app] [info] Success: [QERTCProgram] TC-BPF hook tc_redirect_traffic already exists for interface ens22 (Ignore: libbpf: Kernel error message))
[2026-09-20 01:21:41.946] [upf_app] [info] [QERTCProgram] TC-BPF 'tc_redirect_traffic' attached to ens22 (ingress, ifindex=6)
[2026-09-20 01:21:41.946] [upf_app] [info] 
[2026-09-20 01:21:41.946] [upf_app] [info]   ┌──────────────────────────────────────────────────────────────────────────────────────────────────────────┐
[2026-09-20 01:21:41.946] [upf_app] [info]   │                                      QoS FLOWS - Session 0x1                                             │
[2026-09-20 01:21:41.946] [upf_app] [info]   ├──────┬─────┬──────────────┬────────────┬────────────┬────────────────────────────────────────────────────┤
[2026-09-20 01:21:41.946] [upf_app] [info]   │ QER  │ QFI │    Class     │ GBR (kbps) │ MBR (kbps) │               Flow Description                     │
[2026-09-20 01:21:41.946] [upf_app] [info]   ├──────┼─────┼──────────────┼────────────┼────────────┼────────────────────────────────────────────────────┤
[2026-09-20 01:21:41.946] [upf_app] [info]   │ 1    │ 1   │ 1:38         │ 160,000,000 │ 200,000,000 │ permit out ip from any to any                      │
[2026-09-20 01:21:41.946] [upf_app] [info]   └──────┴─────┴──────────────┴────────────┴────────────┴────────────────────────────────────────────────────┘
[2026-09-20 01:21:41.946] [upf_app] [info] 
[2026-09-20 01:21:41.946] [upf_app] [info] 
[2026-09-20 01:21:41.946] [upf_app] [info]   ┌───────────────────────────────────────────────────┐
[2026-09-20 01:21:41.946] [upf_app] [info]   │           QoS ENFORCEMENT COMPLETED               │
[2026-09-20 01:21:41.946] [upf_app] [info]   │      Session 0x1: 1 QoS Flow(s) configured        │
[2026-09-20 01:21:41.946] [upf_app] [info]   └───────────────────────────────────────────────────┘
[2026-09-20 01:21:41.946] [upf_app] [info] 
[2026-09-20 01:21:41.946] [upf_app] [warning] Session 0x1 has 2 uplink TEIDs, but PDU session map stores only primary TEID 0x3
[2026-09-20 01:21:41.946] [upf_app] [info] [eBPF] Modify Pipeline - Pipeline modified for session 0x1 with 4 PDRs (2 uplink TEIDs, 1 downlink TEIDs)
[2026-09-20 01:21:41.946] [upf_app] [info] [N4] Update Session: seid 0x1 - eBPF data-path pipeline updated successfully
[2026-09-20 01:21:41.946] [upf_app] [info] [N4] Session Modification: seid 0x1 - Completed successfully [Status: Session updated]
```
Looking at the console log of the `nr-ue` command, UE has been assigned the IP address `10.45.0.2` from Open5GS 5GC.
```
[2026-09-20 01:21:42.153] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.45.0.2] is up.
```
Just in case, make sure it matches the IP address of the UE's TUNnel interface.
```
# ip addr show
...
5: uesimtun0: <POINTOPOINT,PROMISC,NOTRAILERS,UP,LOWER_UP> mtu 1400 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none 
    inet 10.45.0.2/24 scope global uesimtun0
       valid_lft forever preferred_lft forever
    inet6 fe80::706:b2a6:46c:5d50/64 scope link stable-privacy 
       valid_lft forever preferred_lft forever
...
```

<a id="ping"></a>

## Ping google.com

Specify the UE's TUNnel interface and try ping.

Please refer to the following for usage of TUNnel interface.

https://github.com/aligungr/UERANSIM/wiki/Usage

<a id="ping_1"></a>

### Case for going through DN 10.45.0.0/16

Run `tcpdump` on VM-DN and check that the packet goes through N6 (ens20).
- `ping google.com` on VM3 (UE)
```
# ping google.com -I uesimtun0 -n
PING google.com (142.250.21.100) from 10.45.0.2 uesimtun0: 56(84) bytes of data.
64 bytes from 142.250.21.100: icmp_seq=1 ttl=106 time=18.6 ms
64 bytes from 142.250.21.100: icmp_seq=2 ttl=106 time=18.6 ms
64 bytes from 142.250.21.100: icmp_seq=3 ttl=106 time=18.0 ms
```
- Run `tcpdump` on VM-DN
```
# tcpdump -i ens20 -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ens20, link-type EN10MB (Ethernet), snapshot length 262144 bytes
01:43:16.780286 IP 10.45.0.2 > 142.250.21.100: ICMP echo request, id 2226, seq 1, length 64
01:43:16.797622 IP 142.250.21.100 > 10.45.0.2: ICMP echo reply, id 2226, seq 1, length 64
01:43:17.781777 IP 10.45.0.2 > 142.250.21.100: ICMP echo request, id 2226, seq 2, length 64
01:43:17.799464 IP 142.250.21.100 > 10.45.0.2: ICMP echo reply, id 2226, seq 2, length 64
01:43:18.783495 IP 10.45.0.2 > 142.250.21.100: ICMP echo request, id 2226, seq 3, length 64
01:43:18.800575 IP 142.250.21.100 > 10.45.0.2: ICMP echo reply, id 2226, seq 3, length 64
```
You could specify the IP address assigned to the TUNnel interface to run almost any applications (iperf3 etc.) as in the following example using `nr-binder` tool.

- `curl google.com` on VM3 (UE)
```
# sh nr-binder 10.45.0.2 curl google.com
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
```
- Run `tcpdump` on VM-DN
```
01:44:41.137666 IP 10.45.0.2.47701 > 142.250.21.139.80: Flags [S], seq 2480186409, win 65280, options [mss 1360,sackOK,TS val 2953587994 ecr 0,nop,wscale 7], length 0
01:44:41.153135 IP 142.250.21.139.80 > 10.45.0.2.47701: Flags [S.], seq 4110012469, ack 2480186410, win 65535, options [mss 1412,sackOK,TS val 4086624513 ecr 2953587994,nop,wscale 8], length 0
01:44:41.154053 IP 10.45.0.2.47701 > 142.250.21.139.80: Flags [.], ack 1, win 510, options [nop,nop,TS val 2953588010 ecr 4086624513], length 0
01:44:41.154053 IP 10.45.0.2.47701 > 142.250.21.139.80: Flags [P.], seq 1:74, ack 1, win 510, options [nop,nop,TS val 2953588011 ecr 4086624513], length 73: HTTP: GET / HTTP/1.1
01:44:41.169998 IP 142.250.21.139.80 > 10.45.0.2.47701: Flags [.], ack 74, win 1050, options [nop,nop,TS val 4086624530 ecr 2953588011], length 0
01:44:41.209128 IP 142.250.21.139.80 > 10.45.0.2.47701: Flags [P.], seq 1:774, ack 74, win 1050, options [nop,nop,TS val 4086624568 ecr 2953588011], length 773: HTTP: HTTP/1.1 301 Moved Permanently
01:44:41.209987 IP 10.45.0.2.47701 > 142.250.21.139.80: Flags [.], ack 774, win 504, options [nop,nop,TS val 2953588066 ecr 4086624568], length 0
01:44:41.210316 IP 10.45.0.2.47701 > 142.250.21.139.80: Flags [F.], seq 74, ack 774, win 504, options [nop,nop,TS val 2953588067 ecr 4086624568], length 0
01:44:41.226635 IP 142.250.21.139.80 > 10.45.0.2.47701: Flags [F.], seq 774, ack 75, win 1050, options [nop,nop,TS val 4086624586 ecr 2953588067], length 0
01:44:41.227464 IP 10.45.0.2.47701 > 142.250.21.139.80: Flags [.], ack 775, win 504, options [nop,nop,TS val 2953588084 ecr 4086624586], length 0
```
Please note that the `ping` tool does not work with `nr-binder`. Please refer to [here](https://github.com/aligungr/UERANSIM/issues/186#issuecomment-729534464) for the reason.
You could now connect to the DN and send any packets on the network using OAI-CN5G-UPF.

---

Now you could work Open5GS 5GC with OAI-CN5G-UPF.
I would like to thank the excellent developers and all the contributors of Open5GS, OAI-CN5G-UPF and UERANSIM.

<a id="changelog"></a>

## Changelog (summary)

- [2026.09.19] Updated to Open5GS v2.8.0 (2026.09.19) and OAI-CN5G-UPF v2.2.1 (2026.09.09).
- [2026.01.23] Rewrote this using OAI-CN5G-UPF built on Ubuntu 24.04.
- [2026.01.18] Initial release.
