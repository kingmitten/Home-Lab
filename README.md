# Proxmox VE Home-Lab

Infrastructure configuration and architecture documentation for a Proxmox VE home lab. This environment is designed for secure network routing, isolated development sandboxes (like a dedicated Minecraft server), and testing backend infrastructure.

## Hardware Specifications

* **CPU:** Intel Core i7-3770
* **RAM:** 16 GB DDR3
* **Network Interface:** Gigabit Ethernet NIC

## Storage Architecture (In Progress)

* **Storage Array:** ZFS Mirrored vDev (RAID 1 equivalent)
* **Architecture Rationale:** ZFS is being implemented over standard hardware RAID 1 to ensure long-term data reliability. Traditional RAID arrays are vulnerable to "bit rot" (silent data corruption). ZFS solves this by using cryptographic checksums for every block of data. If ZFS detects a flipped bit on Disk A, its self-healing architecture will automatically pull the clean data from Disk B and overwrite the corrupted block in real-time.

