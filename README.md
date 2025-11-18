# Home-Lab-pfSense-Firewall-Security-Project


### 1. OBJECTIVES

● Stand up a pfSense firewall in VirtualBox with separate WAN (bridged)  
  and LAN (internal) segments.  
● Place Kali Linux on the WAN side to simulate an external attacker.  
● Place Ubuntu Desktop on the LAN side as a victim host.  
● Demonstrate DoS traffic using **hping3** and analyze it using **Wireshark**.  
● Mitigate the attack using pfSense firewall rules and verify via logs.


### 2. TOPOLOGY & IP PLAN

| Device / Segment        | Adapter        | IF     | IP / Subnet                | Role                         |
|-------------------------|----------------|--------|----------------------------|------------------------------|
| Home LAN                | Host Network   | —      | 192.168.0.0/24             | Internet source              |
| pfSense WAN             | Bridged        | vtnet0 | 192.168.0.254/24 (GW .1)   | WAN firewall interface       |
| Kali Linux (Attacker)   | Bridged        | eth0   | 192.168.0.200/24           | External attacker            |
| Lab LAN (Internal)      | Internal Net   | —      | 192.168.1.0/24             | Internal isolated LAN        |
| pfSense LAN             | Internal Net   | vtnet1 | 192.168.1.1/24             | LAN Gateway / DHCP / DNS     |
| Ubuntu (Victim)         | Internal Net   | eth0   | 192.168.1.100/24           | Victim machine               |


### 3. PREREQUISITES

● **VirtualBox ≥ 7.x installed** (Windows / macOS / Linux)  
    Visit:https://www.virtualbox.org/wiki/Downloads
  - Needed to run pfSense, Kali, Ubuntu  
  - Supports bridged & internal network modes

● **Required ISO images** (latest versions recommended):
  - **pfSense-CE-2.7.x-amd64.iso** → Firewall / Router   Visit:https://www.pfsense.org/download/
  - **kali-linux-current-amd64.iso** → Attacker system   Visit:https://www.kali.org/get-kali/#kali-installer-images
  - **ubuntu-22.04-desktop-amd64.iso** → Victim machine  Visit:https://ubuntu.com/download/desktop

● **Minimum Host Resources**
  - **8 GB RAM**  
  - **40 GB Disk Space**

  Resource breakdown:
  - pfSense → 2 GB RAM / 20 GB Disk  
  - Kali Linux → 2 GB RAM / 15 GB Disk  
  - Ubuntu → 2 GB RAM / 15 GB Disk  

● **Administrator Rights Required**
  - Create bridged network interfaces  
  - Allow guest OS to access physical network  
  - Apply VirtualBox networking changes  

### 4. LAB ENVIRONMENT SETUP

This section covers creating and configuring:

1. pfSense (Firewall)  
2. Kali Linux (Attacker)  
3. Ubuntu Desktop (Victim)  

### 4.1 CREATE pfSense VM

● OS: FreeBSD 64-bit  
● CPU / RAM: 2 vCPU, 2 GB RAM  
● Disk: 20 GB VDI  

**Network Setup:**
- Adapter 1 → **Bridged** (WAN)  
- Adapter 2 → **Internal Network: LabNet** (LAN)

**Steps:**
1. VirtualBox → New → pfSense  
2. Select FreeBSD (64-bit)  
3. Assign RAM + CPU  
4. Create 20 GB disk  
5. Network → Configure adapters  
6. Storage → Attach pfSense ISO  
7. Start installation  

### 4.2 CREATE KALI VM (Attacker)

● OS: Debian 64-bit  
● CPU/RAM: 2 vCPU, 2 GB  
● Disk: 15 GB  

Network:
- Adapter 1 → **Bridged**

**Steps:**
1. Create Debian 64-bit VM  
2. Allocate resources  
3. Attach Kali ISO  
4. Install Kali  
5. Remove ISO  

### 4.3 CREATE UBUNTU VM (Victim)

● OS: Ubuntu 64-bit  
● CPU/RAM: 2 vCPU, 2 GB  
● Disk: 15 GB  

Network:
- Adapter 1 → **Internal Network: LabNet**

**Steps:**
1. Create VM  
2. Allocate resources  
3. Attach ISO  
4. Install OS  
5. Remove ISO  

### 5. pfSense SETUP

● Install pfSense with default settings.  
● Assign interfaces:

```
vtnet0 = WAN  
vtnet1 = LAN  
```

● WAN uses **DHCP**, pfSense gets IP from home router.

### 5.1 Access pfSense from WAN

Disable pfSense firewall temporarily:

### **COMMAND:**
```
pfsense -d
```

**What it does:**  
• Disables pfSense packet filtering to allow WAN GUI access.

Log in via WAN IP.  
Default credentials:

```
Username: admin  
Password: pfsense
```

Add firewall rule to allow WAN GUI access:

```
Firewall ▸ Rules ▸ WAN ▸ Add
Protocol: TCP
Port: 443
Action: Pass
Source: Home Network
```

### 5.2 Configure LAN DHCP

Navigate:  
**Services ▸ DHCP Server ▸ LAN**

Settings:
```
Enable: ✓
Range: 192.168.1.100 – 192.168.1.199
DNS: 192.168.0.1
```

Apply changes.

### 5.3 Allow Kali Access to Internal LAN (Temporary)

Add WAN rule in pfSense:

```
Action: Pass
Protocol: Any
Source: <Kali IP>
Destination: <Ubuntu IP>
Description: Allow Kali access
```

This is required for the DoS demonstration.

### 6. Kali Linux Setup (Attacker)

Check IP address:

### **COMMAND:**
```
ifconfig
```

**What it does:**  
• Displays Kali’s IP address & interface info.

Add route to reach internal LAN:

### **COMMAND:**
```
sudo ip route add 192.168.1.0/24 via <PFSENSE_WAN_IP>
```

**What it does:**  
• Tells Kali to forward packets for 192.168.1.x via pfSense WAN.

### 7. Ubuntu Setup (Victim)

Ubuntu receives:
- IP: **192.168.1.100**  
- Gateway: **192.168.1.1**  
- DNS: **192.168.1.1**  

Verify network:

### **COMMAND:**
```
ifconfig
```

### **COMMAND:**
```
ping -c3 google.com
```

**What it does:**  
• Confirms DNS + internet connectivity.

### 8. Launching the DoS Attack (hping3)

Install Wireshark:

### **COMMAND:**
```
sudo apt update && sudo apt install -y wireshark
```

Run Wireshark:

### **COMMAND:**
```
sudo wireshark &
```

### **ICMP Flood**
```
sudo hping3 -1 --flood 192.168.1.100
```

### **SYN Flood**
```
sudo hping3 --flood -S -p 80 192.168.1.100
```

**What they do:**  
• Send rapid ICMP or SYN packets to overwhelm Ubuntu.

### 9. Block the Attack in pfSense


Create firewall rule:

```
Firewall ▸ Rules ▸ WAN ▸ Add (TOP)
Action: Block
Source: <Kali IP>
Destination: 192.168.1.100
Log: Enabled
Description: Block Kali DoS
```

**Effect:**  
• DoS traffic stops immediately  
• Logs show blocked packets  

### 10. Wrap-Up & Next Steps

● Built a pfSense-based isolated security lab  
● Performed DoS attack simulations  
● Analyzed traffic using Wireshark  
● Mitigated attack via pfSense firewall  
● Configured VirtualBox networking (Bridged & Internal)  
● Ensured segmentation between attacker and victim  

