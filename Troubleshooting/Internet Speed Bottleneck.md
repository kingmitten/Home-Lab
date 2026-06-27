# Root Cause Analysis: Internet speed capped to 2.5MB/s

## Summary

* **Issue:** WAN speed to my homelab had been significantly bottlenecked and capped to 2.5MB/s on a Gigabit Ethernet connection.
* **Fix:** I had removed the conflicted config file that had been blacklisting the driver r8169, and I had removed the unstable driver r8168.
* **Impact:** I had gotten back my full gigabit fiber connection, boosting my WAN speed from 2.5MB/s to 110MB/s while also seeing the physical limits.

## Environment

* **Server:** Debian-based Proxmox VE hypervisor
* **Interface:** Realtek Gigabit Ethernet NIC
* **Physical link:** Wired via Cat5e directly into the router


* **Client:** Windows 11 gaming PC
* **Physical link:** Connected via a Powerline AV-1000 adapter to the router



## Symptoms

1. **Bandwidth throttling:** Incoming network traffic constantly being capped at 2.5MB/s.
2. **Interface instability:** Physical network interface frequently dropped offline and had failed to start up on system boots.
3. **Misconfig:** Prior troubleshooting was based on outdated Reddit and documentations that had incorrectly identified the native Linux driver 'r8169' as faulty, which had resulted in blocking the driver 'r8169' from starting and replacing it to use 'r8168', which had introduced server instability on the kernel.

## Diagnostic and Testing

### Step 1: Physical Link

The link speed between the router and network card was inspected using `ethtool`:

* **Commands used:**
* `apt install ethtool -y`
* **Why?** I did not have this tool downloaded and installed to my system.


* `ethtool nic0 | grep -i speed`
* **Why?** This command allows me to see the current network connection speed for my network interface. The pipe was used to ensure I get the necessary information rather than a wall of text.




* **Result:** `Speed: 1000Mb/s`
* **Deduction:** The physical layer has a full gigabit connection.

### Step 2: Network Software vs Disk Storage

To prove that the 2.5MB/s was not caused by slow disk write speeds, an external 100MB test file was pulled using the `wget` command, routing the download straight to the system's null device.

* **Commands used:**
* `wget -O /dev/null [https://proof.ovh.net/files/100Mb.dat](https://proof.ovh.net/files/100Mb.dat)`


* **Deduction:** By piping the download directly to `/dev/null/` (a black hole that discards the data), this had allowed the physical storage to be bypassed. However, the speed still remained capped at 2.5MB/s, proving a software or driver issue.

### Step 3: Local Networking

To map out the internal network capability, I had used `iperf3` to run a local test between the Linux server and the Windows client.

* **Commands used:**
* `# server sided`
* `iperf3 -s`
* **Why?** To start to listen to incoming network traffic.


* `# client sided`
* `iperf3 -c 192.168.x.xx`
* **Why?** This command was used to test the bandwidth between my computer and my Linux server by sending the data to the listening server at 192.168.x.xx.





## Root Cause Analysis

The root cause was a driver mismatch. The manually compiled r8168 had failed to handle packages effectively, resulting in heavy packet degradation. Furthermore, the manually generated system blacklist prevented the stable native driver r8169 from starting.

## Resolution

To restore the stable network state, the following steps and commands were used:

1. **Locate the conflict:** I had searched for the module config files to find the active driver blacklist rule using the command:
* `grep r8169 /etc/modprobe.d/*`
* **Why?** I had used this command to look for the keyword 'r8169' from the `/etc/modprobe.d` directory to find the config that is blacklisting the driver r8169 from starting.


2. **Remove the blacklist:** I had permanently deleted this file, allowing the driver r8169 to start.
* `rm /etc/modprobe.d/blacklist-r8169.conf`


3. **Rebuild the initramfs:** This was used to regenerate the Linux boot image and ensure that the kernel had initialized the native driver on the next boot cycles.
* `update-initramfs -u`


4. **Reboot**

## Verification

Following the reboot, the physical link had directly been mounted and bounded to the virtual bridge interface.

A final WAN test was conducted:

* `wget -O /dev/null [https://proof.ovh.net/files/100Mb.dat](https://proof.ovh.net/files/100Mb.dat)`

| Metric | Pre-fix | Post-fix |
| --- | --- | --- |
| WAN download speed | 2.5MB/s | 110MB/s |
| Boot status | Crashes on reboot, unable to connect to the internet at times | More stable, can now connect and have full speed |
| Link | 1000Mbps | 1000Mbps |
