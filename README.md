# Home-Lab
Infrastructure configuration and architecture documentation for a Proxmox VE home lab. Features a ZFS RAID 1 NAS, secure network routing , an isolated Minecraft server and custom development sandboxes.


## Hardware Specifications
* CPU : i7-3770
* RAM : 16 GB of DDR 3
* Network Storage Array : ZFS 
  * Why: This is done to help with long term reliability as this contains extra features compared to the RAID 1 system ZFS contains things like self healing data RAID 1 systems are prone to bit rot where the data silently corrupts. ZFS uses a mathematical system which is checksum to store data for example if disk a flips a bit ZFS can detect that and copy the original bit from disk B and overwrite the corrupted block 
* Network Interface : standard gigabit port
