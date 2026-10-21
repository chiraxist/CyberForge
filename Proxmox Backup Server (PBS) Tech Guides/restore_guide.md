# Restore Guide

## Before restoring

1. Connect the drive and attach it to the PBS VM
   - PVE GUI → PBS VM → Hardware
2. On the PBS VM, run:

   ```bash
   lsblk
   ```

   Confirm `sdb1` shows up.
3. If the datastore is showing as offline, mount it with:

   ```bash
   proxmox-backup-manager datastore mount main-backup
   ```

   or use the mount button in the PBS GUI.
4. Check that Datastore → Content loads and lists the `vm/10x` and `host/proxmox` groups.

## To restore a VM

1. In the PVE GUI, go to Datacenter → Storage → click your PBS storage → Backups
2. Expand the VM and pick the snapshot you want
3. Click **Restore**
4. Choose the target storage for the disk. For the VMID, keep the original to replace the existing VM (stop it first), or pick a new one to restore alongside it.
5. Tick **Unique** if you're restoring next to the original, so the copy gets new MAC addresses and doesn't clash.
6. Click **Restore**
7. The disk and the VM's config (`qemu-server.conf`) both come back from the one snapshot.

## Restoring host config (`/etc/pve`, `/etc/network`)

1. Run the following commands on the PVE node to list the host snapshots:

   ```bash
   proxmox-backup-client snapshot list host/proxmox \
     --repository host-backup@pbs@192.168.0.207:main-backup
   ```

2. Restore into a scratch folder, never straight over `/etc/pve`:

   ```bash
   proxmox-backup-client restore host/proxmox/<timestamp> etc-pve.pxar /root/restore-etc-pve \
     --repository host-backup@pbs@192.168.0.207:main-backup

   proxmox-backup-client restore host/proxmox/<timestamp> etc-network.pxar /root/restore-etc-network \
     --repository host-backup@pbs@192.168.0.207:main-backup
   ```

3. To skip the password prompt:

   ```bash
   export PBS_PASSWORD_FILE=/root/.pbs-secrets/pbs-password
   ```

   Use the exact timestamp from the list output.
4. Compare and copy back only what you need, for example:

   ```bash
   diff -r /root/restore-etc-pve /etc/pve
   cp /root/restore-etc-pve/storage.cfg /etc/pve/storage.cfg
   ```

5. If `host-backup@pbs` is refused, use `root@pam` in the repository string instead.

## If the whole PVE server dies

1. Reinstall PVE, ideally with the same hostname and IP
2. Rebuild PBS and attach the external drive
3. In the PBS GUI, add a datastore pointing at the existing drive. Under Advanced, tick **Reuse existing datastore**.
4. Your PBS users and permissions **won't** come back. Log in as `root@pam` on the new PBS and recreate `host-backup@pbs` later if wanted.
5. A new PBS has a new fingerprint. In PVE, add PBS storage under Datacenter → Storage → Add → Proxmox Backup Server with the new fingerprint.
6. Restore the VMs from the GUI as above
7. Restore the host config to a scratch folder and use it as a reference to recreate storage definitions, users and network settings.

*Note:* Restoring from backup recovers the VMs themselves, including their disks, configs and the IPs, hostnames, and passwords set inside them. The Proxmox host and PBS server are not restored automatically. After a rebuild, you must set up again: the PVE and PBS installs, their IPs/hostnames, PBS users and passwords, the datastore (reusing the existing drive) and the PBS storage link in PVE. The backed-up `/etc/pve` and `/etc/network` files can be used as a reference for these settings.
