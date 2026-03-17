# 5G O-RAN 7.2x Integration, Deployment and Performance Evaluation

[![IEEE Paper](https://img.shields.io/badge/IEEE%20ICC%202026-Accepted-green)](https://icc2026.ieee-icc.org/)

This repository contains the configuration files and integration documentation used for deploying a 5G New Radio (NR) standalone network using the [srsRAN Project](https://github.com/srsran/srsRAN_Project) (now [OCUDU](https://gitlab.com/ocudu/ocudu)) gNB stack with a **Benetel RAN650 O-RU** as the radio front-end, following the O-RAN fronthaul (Open Fronthaul) interface specification. The deployment was set up and evaluated at the **Commonwealth Cyber Initiative (CCI) xG Testbed** at Virginia Tech, Arlington, Virginia, USA. This work was accepted for publication:

>**A. Tripathi, F. Bashar, Madhura Adeppady, A. Da Silva and S. F. Midkiff, Varun Singh 
"O-RAN 7.2x Deployment: Insights from Laboratory Integration to Field Trials,"  
IEEE International Conference on Communications (ICC), Glasgow, Scotland, 2026**  


## Repository Structure

```
.
├── 20MHz_Configs/           # 20MHz Bandwidth Configurations
│   ├── ran650_cu.yml        # CU-CP/CU-UP configuration
│   ├── ran650_du.yml        # DU configuration with O-RAN fronthaul
│   └── ru_config.cfg        # Benetel RAN650 RU configuration
│
├── 40MHz_Configs/           # 40MHz Bandwidth Configurations
│   ├── ran650_cu.yml        # CU-CP/CU-UP configuration
│   ├── ran650_du.yml        # DU configuration with O-RAN fronthaul
│   └── ru_config.cfg        # Benetel RAN650 RU configuration
│
├── integration.md           # Step-by-step system preparation, O-RU integration, and RF Tuning.
├── startup.md               # Step-by-step bring-up procedure for 5G Operation.
└── README.md
```

### Configuration Files

Each bandwidth folder (`20MHz_Configs` and `40MHz_Configs`) contains three configuration files:

- **`ran650_cu.yml`** — Configures the srsRAN Open Central Unit (O-CU-CP and O-CU-UP).
- **`ran650_du.yml`** — Configures the srsRAN Open Distributed Unit (O-DU).
- **`ru_config.cfg`** — Configures the Benetel RAN650 Open Radio Unit (O-RU) directly.

### Integration Documentation

- **`integration.md`** — A detailed, step-by-step guide covering the system preparation to integrate and tune the Benetel RAN650 O-RU with the srsRAN O-DU over the O-RAN Open Fronthaul interface. This includes network configuration, DPDK setup, synchronization, and troubleshooting notes based on our hands-on experience.


### Startup Documentation

- **`startup.md`** — The full procedure for running the 5G Network using the Benetel RAN650 O-RU, srsRAN O-DU/O-CU, and Open5GS Core. 

---

## System Overview

The provided configurations and documentation have been tested and validated with the below hardware/software setup. Any variation to this may require modifications to the configuration or deployment procedure.

<img width="413" height="480" alt="Specs" src="https://github.com/user-attachments/assets/6c076adf-ed8e-442f-8b09-311fe8b472e2" />
<img width="374" height="341" alt="image" src="https://github.com/user-attachments/assets/d4254964-af04-4383-9230-cc0eedca6bdd" />

---

## How to Use This Repository

### 1. Prerequisites

Before using these configuration files, ensure the following are in place:

- A server running **Ubuntu 22.04** with a real-time kernel (recommended)
- **srsRAN Project** installed and built (see [srsRAN documentation](https://docs.srsran.com/projects/project))
- **DPDK** installed and configured for your NIC
- **Open5GS** core network deployed and reachable
- **Benetel RAN650** radio unit powered on, synchronized (PTP/SyncE), and accessible over the fronthaul network

### 2. Configuration Adjustments

Before deploying, update the following site-specific parameters in the config files:

**In `ran650_cu.yml`:**
- `amf.addr` and `amf.bind_addr` — set to your AMF IP address
- `plmn`, `tac`, `sst` — set to match your network's PLMN and tracking area

**In `ran650_du.yml`:**
- `ru_mac_addr` — set to the MAC address of the port of your Benetel RAN650 connected to the Fronthaul
- `du_mac_addr` — set to the MAC address of your DU server's fronthaul NIC (or the VF associated with it if using SR-IOV)
- `network_interface` — set to the PCI address of your fronthaul NIC for DPDK
- `hal.eal_args` — update the PCI address and CPU core affinity to match your hardware for DPDK

**In `ru_config.cfg`:**
- `c_plane_du_mac` / `u_plane_du_mac` — set to your DU's fronthaul NIC MAC address (or the VFs associated with them if using SR-IOV, since we used a single connection for C/U Plane, we had identical C/U MAC addresses)
- `centre_frequency_hz` — adjust if deploying on a different frequency

> **Note:** MAC addresses and IP addresses in the published configuration files have been anonymized. Replace all placeholder values with your actual hardware addresses before deployment.

### 3. Running the 5G Stack

Refer to **`startup.md`** for the complete step-by-step bring-up procedure. At a high level, the startup order is:

1. Start the **Open5GS** core network
2. Power on and verify synchronization of the **Benetel RAN650** O-RU
3. Start the **srsRAN CU** (`srscu` binary with `ran650_cu.yml`)
4. Start the **srsRAN DU** (`srsdu` binary with `ran650_du.yml`)
5. Verify fronthaul connectivity and UE attachment via gNB console metrics

---

## Related Work

This repository builds on our earlier work benchmarking SDR-based 5G deployments with srsRAN and NI USRP hardware (X310, N310, B210):

> A. Tripathi, F. Bashar, M. R. Chowdhury, A. Da Silva and S. F. Midkiff,
> "Benchmarking Software Defined Radio Based 5G Deployments With srsRAN: Lessons Learned,"
> *2025 IEEE Wireless Communications and Networking Conference (WCNC)*, Milan, Italy, 2025, pp. 1–6.
> [https://doi.org/10.1109/WCNC61545.2025.10978285](https://doi.org/10.1109/WCNC61545.2025.10978285)

The dataset from that study is available at: [https://github.com/CCI-NextG-Testbed/OTA_dataset_5G_srsRAN](https://github.com/CCI-NextG-Testbed/OTA_dataset_5G_srsRAN)

---

## Citation

>**A. Tripathi, F. Bashar, Madhura Adeppady, A. Da Silva and S. F. Midkiff, Varun Singh 
"O-RAN 7.2x Deployment: Insights from Laboratory Integration to Field Trials,"  
IEEE International Conference on Communications (ICC), Glasgow, Scotland, 2026**  

This repository accompanies a paper accepted to the **IEEE ICC 2026 Workshop on Next-Generation Open and Programmable Radio Access Networks**, to be presented on May 24, 2026 in Glasgow, Scotland. The paper has been accepted but is not yet published in IEEE Xplore. A full citation and DOI will be added here upon official publication following the conference. If you use these configurations or integration procedures in your research in the meantime, please reach out to the authors (contact below) for the appropriate reference.

---

## Contact

For questions regarding this repository, the configurations, or the integration procedure, please contact:

- **Asheesh Tripathi**: asheesh@vt.edu
- **Fahim Bashar**: fahimbashar@vt.edu

---

We hope this repository helps researchers and practitioners deploy and benchmark O-RAN-compliant 5G networks using commercial O-RUs and open-source RAN stacks. 
