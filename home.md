# CTF Challenge: Game of Thrones 1

## 1. Introduction and Objectives
Capture The Flag (CTF) challenges are educational security simulations where practitioners test their penetration testing skills by identifying and exploiting vulnerabilities within a target system. The ultimate goal is to "capture" hidden text files known as "flags".

This report documents the security assessment of the vulnerable virtual machine **Game of Thrones (GOT) 1**, a milestone-based CTF hosted on VulnHub. While the machine includes 7 main flags and 4 extra ones, for the purpose of this academic demo **I focused exclusively on the 7 main flags**. This walkthrough demonstrates a complete attack kill chain, compromising seven thematic kingdoms by exploiting distinct network services, misconfigurations, and cryptographic flaws.

---

## 2. Lab Setup & Threat Model

### Environment Configuration
The testing environment was completely virtualized and isolated using **Oracle VirtualBox**. It consists of two virtual machines connected via **Bridge Mode** to simulate an insider threat scenario:

| Machine | Operating System / Base | Specifications | Purpose |
| :--- | :--- | :--- | :--- |
| **Attacker** | Kali Linux | Default | Network scanning, exploitation, and decryption |
| **Target** | Game of Thrones 1 | 1 vCPU / 1512 MB RAM | Vulnerable host to compromise |

### Threat Model Capabilities
The threat model defined for this assessment assumes the following threat actor capabilities:
1. **Network Placement:** The attacker is located within the same local network segment (LAN) as the target and can transmit/intercept IP packets.
2. **Initial Access:** The attacker can initially communicate only with publicly exposed services (specifically standard HTTP port 80 and SSH port 22).
3. **Ultimate Objective:** Achieving full system compromise (`root` privileges) and exfiltrating all 7 main flags.

---

## 3. Attack and Kingdom Compromise Walkthrough

### Initial Reconnaissance and Network Scanning
To map the network and discover the target, I verified my Kali Linux local IP address using `ip a`, identifying it as `192.168.1.58/24`. Next, I performed a ping scan across the subnet to locate active hosts:

```bash
sudo nmap -sn 192.168.1.0/24