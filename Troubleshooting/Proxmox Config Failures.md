# Root Cause Analysis: Proxmox NAS Build & Config Failures

## Summary 
* **Issue:** The deployment of NextcloudPi had many problems that I had encountered. There were multiple routing and configuration failures, including a broken installation script, blocked UI access, and being unable to access the NAS using the SMB protocol.
* **Fix:** The installation was fixed using an updated community script, the Apache virtual hosts were reconfigured to give the correct directory, and Samba was mapped to the unprivileged LXC UID/GID with the correct ZFS dataset.
* **Impact:** Having a fully ZFS-backed private cloud that allows for both a web UI and Windows-accessible SMB file sharing.

## Environment

* **Server:** Proxmox VE hypervisor (Server: niko, IP: 192.168.x.x)
* **Container:** Unprivileged Debian 12 LXC running NextcloudPi
* **Storage:** 1TB ZFS mirrored pool named nas-storage
* **Client:** Windows system used for testing SMB File Explorer access

## Symptoms 

1. **Installation failure:** The initial Nextcloud installation script was silently bringing me back to the prompt without fully executing.
2. **Web UI inaccessibility:** The NextcloudPi admin panel on the port 4443 had failed to start and connect, and port 443 had displayed a forbidden internal error instead of a login page.
3. **Login failures:** The user passwords did not work after the initial setup was completed.
4. **Network share errors:** Windows was unable to access the SMB share `\\192.168.x.x\NAS` and a local connection test had returned an `NT_STATUS_BAD_NETWORK_NAME` error.

## Diagnostic

### Step 1: Installer failure 
To determine why the script had silently returned to the prompt, `wget` was executed without quiet mode to reveal it was an HTTP error.

* **Commands used:**
  * `wget -O - https://github.com/tteck/Proxmox/raw/main/ct/nextcloud.sh`
  * **Why:** The initial script had run in silent mode `-q`, this made it so I could not see any errors if any had occurred.
* **Result:** I had found that there was a 404 error which had indicated that the script was dead and/or inactive.
* **Deduction:** Proxmox DNS and networking were fully operational; the script URL used was dead.

### Step 2: Web server port routing
To locate the cause of the forbidden pages and blocked UI, the Apache listening ports and the virtual hosts were inspected.

* **Commands used:**
  * `pct exec 100 -- bash -lc "ss -ltnp | grep -E ':(80|443|4443)'"`
  * **Why:** To check if the Apache ports were actually listening inside the container.
  * `pct exec 100 -- apache2ctl -S`
  * **Why:** To list the current configured virtual hosts and see where the web traffic was being routed to.
* **Result:** Apache was listening on port 80 and 443, but no listener existed for 4443. As well as the port 443 was being handled by the `ncp-activation.conf` instead of the main Nextcloud document root.
* **Deduction:** The Nextcloud activation page had remained active on port 443 which had blocked the main Nextcloud interface from starting, as well as the admin config was present but had not been activated.

### Step 3: SMB path debugging 
To locate and isolate the Windows SMB connection issue, some tests were done locally on the Proxmox host to bypass the network and firewall.

* **Commands used:**
  * `smbclient //127.0.0.1/NAS -U smbuser`
  * **Why:** It was done to test the SMB connection locally on the Proxmox to determine if the issue was related to Windows or the Samba config.
  * `testparm -s`
  * **Why:** To dump and validate the actively loaded Samba block.
* **Result:** The local connection failed with the error `tree connect failed: NT_STATUS_BAD_NETWORK_NAME`. Furthermore, `testparm` revealed that the share path was incorrectly set to `/nas-storage/data/ncp/files`.
* **Deduction:** The configured path in SMB config was missing the `nextcloud-data` layer, causing Samba to fail to broadcast the path.

## Root Cause Analysis 
The deployment of NextcloudPi I had faced with 3 root causes across the building phase. First, the initial Nextcloud installation had failed due to the URL being a dead or inactive GitHub URL returning a 404 HTTP error. Second, the NextcloudPi web UI was inaccessible as the Apache configuration was stuck in the `ncp-activation` profile, which had caused the failure to route the port 443 to the actual Nextcloud document root. And finally, the SMB was inaccessible as the `smb.conf` file had contained the incorrect directory path and had also lacked the `nextcloud-data` layer, as well as lacking the specific UID/GID mapping required to interface safely with an unprivileged LXC container's filesystem.

## Resolution

To restore the functionality of Nextcloud, the following steps had been taken: 

1. **Updated the installation script:** I had used the updated community script.
   * `bash -c "$(wget -qLO - https://github.com/community-scripts/ProxmoxVE/raw/main/ct/nextcloudpi.sh)"`
2. **Fixed the Apache routing:** I had disabled the broken activation site and enabled the correct Nextcloud and admin virtual hosts.
   * `pct exec 100 -- bash -lc "a2ensite ncp.conf && systemctl reload apache2"`
   * `pct exec 100 -- bash -lc "a2dissite ncp-activation.conf && a2ensite 001-nextcloud.conf && systemctl reload apache2"`
3. **Migrated the storage to ZFS:** Bound a ZFS dataset to the container and migrated the Nextcloud data over.
   * `pct set 100 -mp0 /nas-storage/nextcloud-data,mp=/mnt/nas-data`
   * `pct exec 100 -- bash -lc "rsync -Aax /opt/ncdata/data/ /mnt/nas-data/data/"`
4. **Mapped the unprivileged SMB permissions:** I had created a host-side user mapping to match the unprivileged container's internal UID/GID.
   * `groupadd -g 100033 nc-wwwdata && useradd -u 100033 -g 100033 -M -s /usr/sbin/nologin nc-wwwdata`
5. **Corrected the Samba configuration:** I had fixed the incorrect path in `/etc/samba/smb.conf` to `/nas-storage/nextcloud-data/data/ncp/files` and applied the force user setting.
   * `systemctl restart smbd`

## Verification

Following the configuration corrections, both Nextcloud and SMB services became fully operational and successfully linked to the shared ZFS dataset.

| Metric | Pre-fix | Post-fix |
| --- | --- | --- |
| **Web UI Access** | Forbidden / Internal error on port 443 | Successful Nextcloud login at `https://192.168.1.41` |
| **Nextcloud Data Store** | Container local storage on `local-lvm` | Safely backed to ZFS dataset `/mnt/nas-data/data` |
| **Windows SMB Connection** | `NT_STATUS_BAD_NETWORK_NAME` | File Explorer access granted to `\\192.168.1.200\NAS` |
| **SMB/Nextcloud Sync** | SMB uploads hidden from Nextcloud UI | Unified visibility via `occ files:scan` cron automation |
