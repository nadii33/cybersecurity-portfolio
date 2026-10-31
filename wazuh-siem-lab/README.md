# SIEM Home Lab: Wazuh

## Summary

Deployed a Wazuh SIEM (manager, indexer, dashboard) in a VirtualBox VM to practice log collection, monitoring and alert analysis. 
Generated simulated failed SSH logins and analysed the resulting alerts in the dashboard. Troubleshot a failed installation (undersized VM, interrupted download, broken package state).

**Résumé :** J'ai installé un SIEM Wazuh (manager, indexer et dashboard) dans une machine virtuelle VirtualBox pour m'entraîner à collecter des logs, 
surveiller un système et analyser des alertes. J'ai simulé des connexions SSH échouées, puis j'ai regardé les alertes dans le tableau de bord. J'ai aussi réglé plusieurs problèmes
d'installation sous Linux : machine trop petite, installation interrompue et paquet cassé.


## Environment

| Item | Value |
|---|---|
| Hypervisor | VirtualBox |
| OS | Ubuntu Server 22.04 LTS (hostname: `wazuh-server`) |
| VM resources (final) | 4 CPUs, 8 GB RAM, 50 GB disk |
| SIEM | Wazuh 4.14, all-in-one install (manager, indexer, dashboard) |
| Log shipper | Filebeat |
| Network | NAT with port forwarding (host 8443 → guest 443) |

Initial VM specs were 2 CPUs, 4 GB RAM and 40 GB disk. These turned out to be too small.

## Installation

1. **Create the VM** in VirtualBox with the specs above and the Ubuntu Server 22.04 ISO. Used plain NAT networking. I first tried an NAT Network to run a second VM as a client, but my machine didn't have enough RAM to run two VMs.

2. **Note the VM's IP address:**
   ```bash
   ip a
   ```

3. **Update the system:**
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

4. **Download and run the Wazuh installer:**
   ```bash
   curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
   sudo bash ./wazuh-install.sh -a
   ```
   When it finishes, it prints the `admin` username and a generated password. 

5. **Add a port forwarding rule** in VirtualBox (Network settings for the network the VM actually uses): Protocol TCP, Host Port 8443, Guest Port 443.

6. **Check that all four services are running:**
   ```bash
   sudo systemctl is-active wazuh-manager wazuh-indexer wazuh-dashboard filebeat
   ```
   Expected output: four lines saying `active`.

7. **Open the dashboard** in the host browser at `https://localhost:8443` and log in with the credentials from step 4.

## Generating test events

The Wazuh agent can't be installed on the same host as the manager (see Troubleshooting, entry 3), so I generated events directly on the server. 
This simulates failed SSH logins for a user that doesn't exist:

```bash
for i in 1 2 3 4 5; do
  ssh -o StrictHostKeyChecking=no -o BatchMode=yes -o ConnectTimeout=2 fakeuser@localhost
done
```

`BatchMode=yes` makes SSH fail immediately instead of prompting, so each attempt is a real failed login.

## Results

After the failed SSH logins, I checked the raw log first. `/var/log/auth.log` shows the "Failed password for invalid user fakeuser" lines.

![Failed SSH logins for fakeuser](https://github.com/user-attachments/assets/dbcaf071-958a-4a54-9606-1379cde161ca)
![Failed password entries in auth.log](https://github.com/user-attachments/assets/909e8250-3368-4715-8b5a-7a2e1062cecf)

Then I opened the Wazuh dashboard. Over the last 24 hours it showed 63 alerts on wazuh-server, and none at level 12 or above.

![Dashboard overview](https://github.com/user-attachments/assets/d76ea85d-42c3-45e2-b444-75f1aed96172)

When I searched for `sshd`, 8 alerts were left. All of them were authentication failures, and all were tagged Brute Force in MITRE ATT&CK.

![sshd dashboard](https://github.com/user-attachments/assets/b0e69a98-91f3-4591-b9b7-d087ca3f915f)
![Alert list](https://github.com/user-attachments/assets/5d169005-8d62-4357-a33d-3d0b01d60805)

Two rules fired, both at level 10:

- Rule 2502: "User missed the password more than one time" (7 alerts)
- Rule 5712: "sshd: brute force trying to get access to the system. Non existent user." (1 alert)

Rule 5712 only fires after 8 failed logins, and its details show the log lines that triggered it. Both rules map to T1110 (Brute Force).

![Rule 5712 details](https://github.com/user-attachments/assets/951b64f2-3b8a-4532-aec9-b165f8681c9f)
![Rule 2502 details](https://github.com/user-attachments/assets/ab25ab52-60c4-4bc1-8ab5-220535c6a372)


## Troubleshooting

### 1. Indexer and dashboard inactive after installation

- **Symptom:** the service check showed `inactive` for `wazuh-indexer` and `wazuh-dashboard`.
- **Investigation:**
  - `sudo tail -n 20 /var/log/wazuh-install.log` showed nothing meaningful.
  - `sudo grep -i error /var/log/wazuh-install.log` returned too many lines, so I limited it with `grep -i -m 5 error`, which showed an `error processing package` message.
  - `grep -i -m 3 -A 2 processing` on the same log showed that port 1515 was already in use by leftover processes from the failed install.
  - `free -h` showed that almost all of the RAM was in use.
  - `lsblk` showed that the VM had a 50 GB disk but Linux was only using about 19.5 GB of it.
- **Cause:** the VM was undersized, and the LVM volume hadn't been extended to use the whole disk.
- **Fix:**
  1. Shut the VM down and increased RAM and disk in VirtualBox.
  2. Grow the partition and the filesystem:
     ```bash
     sudo apt install -y cloud-guest-utils
     sudo growpart /dev/sda 3
     sudo pvresize /dev/sda3
     sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
     df -h /
     ```
  3. `df -h /` then showed 37 GB available.
  4. Removed the old install log and re-ran the installer with `sudo bash ./wazuh-install.sh -a -o`.

### 2. Interrupted installation and broken manager package

- **Symptom:** the second install attempt finished the manager step in about 2 seconds, which was too fast to be real. It then printed
   `wazuh-keystore: No such file or directory` and finally `The Wazuh manager package could not be removed`.
- **Cause:** the download was interrupted because my host computer went to sleep while I was away. The manager's files were never unpacked,
   but a broken package was left behind.
- **Fix:** run the commands one at a time:
  ```bash
  sudo dpkg --configure -a                    # finish interrupted package operations
  sudo dpkg --purge --force-all wazuh-manager # remove the broken package and its config
  sudo rm -rf /var/ossec                      # delete leftover Wazuh files
  sudo apt --fix-broken install -y            # repair any half-finished package state
  dpkg -l 'wazuh*'                            # confirm nothing Wazuh-related remains
  sudo bash ./wazuh-install.sh -a -o          # reinstall
  ```
  After this, all four services showed `active`.
- **Note:** `--force-all` and `rm -rf` are acceptable in a throwaway lab, but I wouldn't use them on a production system.

### 3. Agent install fails: conflicting packages

- **Symptom:** the dashboard's "deploy new agent" command failed with `conflicting packages: not installing wazuh-agent`.
- **Cause:** `wazuh-manager` and `wazuh-agent` can't be installed on the same host.
- **Workaround:** the manager host already monitors itself, so I generated test events directly on the server.
- **Next step:** with more RAM, run a second VM as a monitored agent.

### 4. SSH simulation asked to confirm the host key

- **Symptom:** the first version of the loop (`ssh fakeuser@localhost`) stopped at `Are you sure you want to continue connecting?`. I answered `no` so it wouldn't save the host key.
- **Fix:** added `-o StrictHostKeyChecking=no -o BatchMode=yes -o ConnectTimeout=2` so each attempt fails immediately without prompting.

### 5. Dashboard not reachable from the host

- **Symptom:** the browser couldn't reach the dashboard after installation.
- **Cause:** I had created the port forwarding rule on the NAT Network instead of on the plain NAT I was actually using.
- **Fix:** moved the rule to the correct network (TCP, host 8443 → guest 443) and opened `https://localhost:8443`.

## What I learned
- Long installs need a stable host - when my laptop went to sleep, the download stopped halfway and left a broken package. I now keep the machine awake during
  installs and know how to repair the package state with `dpkg`.
- A VM on a NAT network is not reachable from the browser by default, so I needed port forwarding to open the Wazuh dashboard.
- Wazuh turns repeated low-level failures into a single higher-level brute-force alert, and I can follow it end to end from the raw log in /var/log/auth.log to
  the dashboard.
- How to read an alert: rule ID, level, groups, MITRE mapping and compliance tags.
- That a virtual disk can be extended inside Ubuntu (LVM) before the system can use it.
