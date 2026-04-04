# UGREEN DXP4800 — Full NAS Setup Guide
### OMV 7 + SnapRAID + MergerFS + Btrfs

---

## Overview

| Component | Role | Filesystem |
|-----------|------|------------|
| NVMe 1TB — partition 1 (~60 GB) | OMV operating system | ext4 (installer managed) |
| NVMe 1TB — partition 2 (~940 GB) | MergerFS write cache | Btrfs |
| HDD Bay 1 — 18 TB Toshiba | **Data drive** (MergerFS pool + SnapRAID data) | Btrfs |
| HDD Bay 2 — 18 TB Toshiba | **Parity drive** (SnapRAID parity only) | ext4 |
| Bays 3 & 4 | Empty — reserved for future expansion | — |

**MergerFS pool** = NVMe cache + HDD Bay 1 (all read/write goes here)  
**SnapRAID** protects Bay 1 using Bay 2 as parity (Bay 2 is never in the MergerFS pool)  
**Usable storage**: ~18 TB data + ~940 GB fast cache

> **Note on future expansion**: When you add drives to Bays 3 and 4, you add them to the MergerFS pool and as SnapRAID data drives. The existing Bay 2 parity drive (18 TB) can protect up to 3 additional 18 TB data drives.

---

## What You Need Before Starting

- USB drive (at least 4 GB)
- A PC to create the installer USB
- Monitor + USB keyboard (for initial BIOS setup and installation)
- Network cable connected to the DXP4800
- Access to your router (to find the NAS IP or set a reservation)

---

## Phase 1 — Create the OMV Installation USB

1. On your PC, download the **OMV 7 ISO** from:  
   `https://www.openmediavault.org/download.html`  
   Get the latest `openmediavault_7.x.x-amd64.iso`.

2. Flash it to your USB drive using **Rufus** (Windows) or **Balena Etcher** (any OS):
   - Rufus: Select the ISO → Partition scheme: **GPT** → Target: **UEFI** → Write
   - Etcher: Just select image → select USB → Flash

---

## Phase 2 — BIOS Setup on the DXP4800

1. Connect monitor and keyboard to the DXP4800.
2. Insert your USB drive.
3. Power on and **press `CTRL + F2`** repeatedly to enter BIOS.
4. In BIOS:
   - Set **Boot Mode** to **UEFI** (not Legacy/CSM)
   - Set **Boot Order**: USB drive first, NVMe second
   - Disable **Secure Boot** (can cause issues with Debian-based installers)
   - Disable **Watchdog** (look under Advanced or Power settings — if left enabled, the system may reboot itself unexpectedly when OMV is idle)
5. Save and exit (`F10`).

---

## Phase 3 — Install OMV 7 with Manual NVMe Partitioning

The installer is a standard Debian installer. Follow these steps carefully — the partitioning step is the most important part.

### 3.1 Language & Network
1. Select your language, location, and keyboard layout.
2. Let it auto-configure the network via DHCP. You'll set a static IP later.
3. Set the **hostname** (e.g., `nas` or `dxp4800`).
4. Leave domain name blank or enter your local domain if you have one.
5. Set a strong **root password** and note it down.

### 3.2 Manual Disk Partitioning (Critical Step) (<-- does not working during setup)

❗ this did not work as described here, the setup won't let you partition while installing. you need to boot into gparted afterwards, shrink the ext4 to whatever size you desire and then create a new btrfs partition with the unallocated space, label "cache", then continue with Phase 4

When you reach **"Partition disks"**:

1. Choose **"Manual"**.
2. You will see your disks. Select the **NVMe drive** (it will show as `nvme0n1` or similar — look for the ~1 TB device, *not* your HDDs).
3. Create a new **GPT partition table** on it (select the disk → "Create new empty partition table" → GPT).
4. Now create three partitions:

   **Partition 1 — EFI**
   - Size: `512 MB`
   - Use as: `EFI System Partition`
   
   **Partition 2 — OS Root**
   - Size: `60 GB`
   - Use as: `ext4 journaling file system`
   - Mount point: `/`
   
   **Partition 3 — Cache (do NOT format yet)**
   - Size: `remaining space` (use "max")
   - Use as: `do not use` ← important, we'll format this ourselves after install

5. The two HDDs should say "do not use" — **leave them completely untouched**.
6. Select **"Finish partitioning and write changes to disk"** → confirm.

### 3.3 Complete the Installation

1. When asked about a swap partition, choose **No** (OMV doesn't need swap on modern hardware with sufficient RAM).
2. Let it install the base system.
3. When asked about the **GRUB bootloader**, install it to the NVMe drive (`/dev/nvme0n1`).
4. After installation completes, remove the USB when prompted and allow the system to reboot.

---

## Phase 4 — First Boot & Initial Configuration

### 4.1 Find the NAS on Your Network

From another computer, open a browser and go to:
```
http://<nas-ip-address>
```
If you don't know the IP, check your router's DHCP client list, or look at the DXP4800's screen if it has one. You can also SSH in:
```bash
ssh root@<ip>
```

### 4.2 Log Into the OMV Web Interface

- URL: `http://<nas-ip>`
- Username: `admin`
- Password: `openmediavault`

**Immediately change the admin password:**  
`User Settings` → `Change Password`

### 4.3 Set a Static IP

This ensures your NAS is always reachable at the same address.

**Option A — Set it in the router (recommended):**  
In your router's admin panel, find the NAS's MAC address in the DHCP client list and set a static DHCP reservation for it. This is the cleanest approach.

**Option B — Set it in OMV:**  
`Network` → `Interfaces` → click on your interface (e.g., `eth0`) → Edit:
- IPv4 Method: `Static`
- Address: e.g., `192.168.1.50`
- Netmask: `255.255.255.0`
- Gateway: your router IP (e.g., `192.168.1.1`)

Save and apply.

### 4.4 Enable SSH

`Services` → `SSH` → Enable → Save → Apply

This lets you run commands on the NAS without a keyboard/monitor attached.

### 4.5 Update the System

`System` → `Update Management` → Check for updates → Install all updates → Apply

Then from SSH, also run:
```bash
apt update && apt full-upgrade -y
```

---

## Phase 5 — Install omv-extras and Plugins

`omv-extras` is a community plugin that unlocks additional packages including SnapRAID and MergerFS plugins.

### 5.1 Install omv-extras

SSH into your NAS as root and run (verify on https://github.com/OpenMediaVault-Plugin-Developers/packages):
```bash
wget -O - https://github.com/OpenMediaVault-Plugin-Developers/packages/raw/master/install | bash
```

Wait for it to finish. You may need to refresh the OMV web UI after this.

### 5.2 Install SnapRAID and MergerFS Plugins

In the OMV web UI:  
`System` → `Plugins`

Search for and **install both**:
- `openmediavault-snapraid`
- `openmediavault-mergerfs`

Click "Install" for each, wait for completion, then click "Apply".

---

## Phase 6 — Check the HDDs with S.M.A.R.T.

Before putting any data on the drives, verify they are healthy and genuinely new. S.M.A.R.T. (Self-Monitoring, Analysis and Reporting Technology) reads diagnostics built into every drive.

### 6.1 Enable S.M.A.R.T. Monitoring in OMV

`Storage` → `S.M.A.R.T.` → `Settings` → Enable → Set **Check Interval** to 3600 and **Power mode** to Standby → Save → Apply.

### 6.2 Run a Short Self-Test on Each Drive

`Storage` → `S.M.A.R.T.` → `Devices` tab

You should see both Toshiba drives listed. For each drive:
1. Select the drive
2. Click **Edit** → check **Monitoring enabled** → run it
3. Click **Scheduled Tasks** → **Create** → Choose the drive, Hour: 1, Day of week: Sunday → **Save**

A short test takes 1–2 minutes per drive. Refresh the page and check the **Self-test logs** tab — the result should show `Completed without error`.

### 6.3 Check S.M.A.R.T. Attributes

For each drive, click on it and go to the **Attributes** tab. The critical values to check are:

| Attribute | ID | What to look for |
|-----------|----|-----------------|
| Reallocated Sectors Count | 5 | Must be **0** — any non-zero value means the drive has already remapped bad sectors |
| Reported Uncorrectable Errors | 187 | Must be **0** |
| Current Pending Sector Count | 197 | Must be **0** — sectors waiting to be remapped |
| Offline Uncorrectable | 198 | Must be **0** |
| Power-On Hours | 9 | Should be very low (under ~50h) for a new drive — a high value means the drive isn't new |

### 6.4 Cross-Check via CLI (not needed because you can see the smartctl log already)

For a full raw dump from the command line:

```bash
# Install smartmontools if not already present
apt install smartmontools -y

# Full attribute report for each drive
smartctl -a /dev/sda
smartctl -a /dev/sdb
```

Look for `SMART overall-health self-assessment test result: PASSED` near the top of each output.

> **If a drive fails or shows any non-zero values in the critical attributes above:** Don't use it. Return it for a replacement before proceeding with the rest of the setup.

---

## Phase 7 — Prepare the NVMe Cache Partition

We left the third NVMe partition unformatted during installation. Now we'll set it up as a Btrfs cache volume.

### 7.1 Identify the Partition

SSH in and run:
```bash
lsblk -f
```

You should see something like:
```
nvme0n1
├─nvme0n1p1   vfat    EFI         ...  /boot/efi
├─nvme0n1p2   ext4    ...             /
└─nvme0n1p3                           (unformatted, ~940GB)
```

Note the partition name (e.g., `nvme0n1p3`).

### 7.2 Format as Btrfs

```bash
mkfs.btrfs /dev/nvme0n1p3 -L "cache"
```

### 7.3 Mount in OMV

Now tell OMV about this filesystem:  
`Storage` → `File Systems` → Click the **►** (Mount an existing file system) button

OMV will scan and list unmounted filesystems. Find the Btrfs partition on your NVMe (~940 GB, labeled "cache") → select it → **Save** → **Apply**.

It will be mounted at a path like:
```
/srv/dev-disk-by-uuid-XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
```
Make a note of this path — you'll need it later. You can find it in `Storage` → `File Systems`.

---

## Phase 8 — Format the Data Drives

The data drive (Bay 1) gets **Btrfs** — its checksumming is what lets SnapRAID detect bitrot at the filesystem level. The parity drive (Bay 2) gets **ext4** — SnapRAID's parity is a single large binary file that SnapRAID checksums internally; Btrfs adds no benefit there and introduces background processes that can interfere with hd-idle spindown.

> ⚠️ This will erase everything on both drives. Make sure you haven't put any data on them.

### 8.1 Identify the Drives

```bash
lsblk -d -o NAME,SIZE,MODEL
```

Your Toshiba drives will appear as `/dev/sda` and `/dev/sdb` (or similar). Confirm with the size (18 TB).

### 8.2 Format the Drives

```bash
# Bay 1 — Data drive: Btrfs for filesystem-level checksumming
mkfs.btrfs /dev/sda -L "data1"

# Bay 2 — Parity drive: ext4, simpler and correct for SnapRAID's single parity file
mkfs.ext4 -L "parity1" /dev/sdb
```

### 8.3 Mount Both Drives in OMV

`Storage` → `File Systems` → click **►** (Mount)

Mount **both** drives, one at a time. OMV will assign each a `/srv/dev-disk-by-uuid-...` path. Note both paths — you'll need them for SnapRAID.

---

## Phase 9 — Configure SnapRAID (seems outdated for the first 4 substeps. requires array and then add the drives)

SnapRAID protects against drive failure and detects bitrot. It is **not** real-time RAID — it takes periodic snapshots. This means you must sync regularly (covered in Phase 15).

### 9.1 Open SnapRAID Configuration

`Services` → `SnapRAID`

### 9.2 Configure Parity Drive

In the **Parity** section, add:
- **Parity file path**: `<parity-drive-mount-path>/snapraid.parity`  
  Replace `<parity-drive-mount-path>` with the actual mount path of Bay 2 (e.g., `/srv/dev-disk-by-uuid-XXXXXX/snapraid.parity`)

### 9.3 Add Data Drive

In the **Drives** section, add:
- **Content file**: Enabled
- **Data drive path**: `<data-drive-mount-path>`  
  This is the mount path of Bay 1

Also add a content file on the parity drive:  
Add a second entry pointing to `<parity-drive-mount-path>/snapraid.content`

And optionally on the NVMe cache:  
`<nvme-cache-mount-path>/snapraid.content`

> Having the content file in multiple locations is recommended — SnapRAID needs at least one to function after a failure.

### 9.4 Configure SnapRAID Settings

Still in `Storage` → `SnapRAID` → `Settings` tab:
- **Exclude files**: add `*.unrecoverable` and `tmp/`
- **Autosave**: `500` (saves content file every 500 GB of processed data — protects against interrupted syncs)
- Everything else can stay at defaults for now.

### 9.5 Run First Sync

`Storage` → `SnapRAID` → click **Sync**

This first sync will take a long time on 18 TB drives (could be several hours). It's computing parity for all existing data (none yet, so it's just initializing — it will be fast now, but longer once data is present).

> **Why running sync directly is safe here (and only here):** The diff script and AIO script both require at least one prior sync to have completed before they can operate — they read the content file that sync produces. On a brand new empty array there is also nothing to lose. This is the one legitimate time to run `snapraid sync` directly. After this, all future syncs go through the diff script configured in Phase 15.

---

## Phase 10 — Configure MergerFS (Two-Pool Setup) (this can all be done in the omv gui)

Two separate MergerFS pools are used. This solves two problems at once: the NVMe always receives writes first, and when data is flushed to the HDDs it is distributed evenly across all of them rather than piling onto one.

```
Clients / Shares
      ↓
  Pool 2: "pool"  (/srv/mergerfs/pool)
  NVMe first → falls through to ↓
  Pool 1: "data"  (/srv/mergerfs/data)
  HDDs, distributed evenly via epmfs
      ↓
  SnapRAID (sees individual HDD drives directly, never the NVMe)
```

**Why two pools?** A single pool cannot independently control "always write to NVMe first" and "distribute evenly across HDDs" — those are two separate concerns. Pool 1 handles HDD distribution. Pool 2 adds the NVMe as a fast front door on top. When you add drives in the future, you only touch Pool 1 — Pool 2 never changes.

---

### 10.1 Create Pool 1 — HDD Data Pool

This pool contains only your HDD data drives. It is what SnapRAID's data ultimately lands on. Shares and applications never talk to this pool directly — it is an internal layer.

`Storage` → `MergerFS` → **+**

| Setting | Value |
|---------|-------|
| Name | `data` |
| Drives | Bay 1 (HDD data drive) only. Add more HDDs here when you expand. |
| Mount point | `/srv/mergerfs/data` |
| Options | See annotated options below |

**Pool 1 options — copy this string:**
```
defaults,allow_other,use_ino,cache.files=off,dropcacheonclose=true,category.create=epmfs,minfreespace=50G,moveonenospc=true,noatime
```

**What each option does (keep this for future reference):**

- `allow_other` — lets non-root users (your NAS user, SMB/NFS daemon) access the mount. Without this, only root can see it. *Search: mergerfs allow_other*
- `use_ino` — uses inode numbers from the underlying drives instead of generating new ones. Prevents inode conflicts between files on different drives. *Search: mergerfs use_ino inode*
- `cache.files=off` — disables MergerFS's own file content cache. The Linux kernel already caches files independently; a second cache layer causes stale data bugs. `off` is the safe conservative choice. *Search: mergerfs cache.files*
- `dropcacheonclose=true` — when a file handle is closed, flush any cached data for it. Pairs with `cache.files` to prevent stale reads. *Search: mergerfs dropcacheonclose*
- `category.create=epmfs` — **write policy: Existing Path, Most Free Space.** If the destination folder already exists on a specific drive, new files go there (keeps related files together). For brand new folders, picks the drive with the most free space. As drives fill over the years, the "most free" winner rotates naturally, giving you even distribution without any manual intervention. *Search: mergerfs epmfs create policy*
- `minfreespace=50G` — never write to a drive with less than 50GB free. Prevents a drive from being completely filled, which causes filesystem errors. Adjust upward if you use very large files. *Search: mergerfs minfreespace*
- `moveonenospc=true` — if a write fails because a drive unexpectedly hits its limit, automatically retry on another drive in the pool instead of returning an error to the application. *Search: mergerfs moveonenospc*
- `noatime` — **do not update the access timestamp when a file is read.** Without this, every file read triggers a metadata write to update the "last accessed" time on the HDD, which wakes sleeping drives unnecessarily and causes extra wear. `noatime` is standard practice for NAS storage. *Search: noatime fstab linux*

Click **Save** → **Apply**. Verify:
```bash
ls /srv/mergerfs/data
```

---

### 10.2 Create Pool 2 — Cached Pool

This is the pool that everything else uses: shares, SMB, NFS, Immich. It puts the NVMe in front of Pool 1 as a write cache. Writes always land on the NVMe first; Pool 1 only receives data after the nightly flush.

Pool 2 stacks on top of Pool 1 (a FUSE mount), which the OMV GUI may not support directly. Set it up as a **systemd mount unit** — this is the safe, OMV-compatible way to do it.

**Step 1 — Find your NVMe cache UUID:**
```bash
blkid /dev/nvme0n1p3
# Look for: UUID="XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX"
```

**Step 1b — Verify the actual Pool 1 systemd unit name:**

The Pool 2 unit must declare Pool 1 as a dependency. But OMV's MergerFS plugin generates the Pool 1 unit name internally — verify it before hardcoding it:

```bash
systemctl list-units | grep mergerfs
```

You should see a unit like `srv-mergerfs-data.mount`. Note the exact name — use it in the `After=` and `Requires=` lines below. If the name differs from `srv-mergerfs-data.mount`, substitute accordingly.

**Step 2 — Create the systemd mount unit:**
```bash
nano /etc/systemd/system/srv-mergerfs-pool.mount
```

Paste this content (substitute your actual NVMe UUID):
```ini
[Unit]
Description=MergerFS Cached Pool
After=srv-mergerfs-data.mount
Requires=srv-mergerfs-data.mount

[Mount]
What=/srv/dev-disk-by-uuid-<nvme-cache-uuid>:/srv/mergerfs/data
Where=/srv/mergerfs/pool
Type=fuse.mergerfs
Options=defaults,allow_other,use_ino,cache.files=off,dropcacheonclose=true,category.create=ff,minfreespace=10G

[Install]
WantedBy=multi-user.target
```

> **Performance note — double FUSE overhead:** Pool 2 is a FUSE filesystem stacked on top of Pool 1, which is also a FUSE filesystem. Every read goes through two FUSE layers. For sequential reads of large files (copying a 10 GB file over SMB) this overhead is negligible. For workloads involving many small files or frequent random access — such as Immich generating thumbnails or running face recognition — this can be a meaningful performance hit. If you notice sluggishness with Immich specifically, that's the likely cause; the tradeoff is accepted in exchange for correct NVMe-first write routing and even HDD distribution. *Search: mergerfs fuse overhead stacking*

> **Filename must exactly match the mount path** with slashes replaced by dashes: mount path `/srv/mergerfs/pool` → filename `srv-mergerfs-pool.mount`. This is a systemd requirement. *Search: systemd mount unit naming*

**Pool 2 options — what each does:**

- `category.create=ff` — **write policy: First Found.** Writes to the first branch in the `What=` list that has space above `minfreespace`. NVMe is listed first, so all writes go there unconditionally. Falls through to Pool 1 (the HDD pool) only when NVMe is nearly full. Unlike `mfs` or `lfs`, this never compares drive sizes — order is the only rule. *Search: mergerfs ff first found policy*
- `minfreespace=10G` — NVMe safety buffer. Stops writing to the NVMe when less than 10GB remains. At that point writes transparently fall through to Pool 1 and its `epmfs` distribution. *Search: mergerfs minfreespace*
- All other options are identical to Pool 1 above.

**Step 3 — Enable and start it:**
```bash
systemctl daemon-reload
systemctl enable srv-mergerfs-pool.mount
systemctl start srv-mergerfs-pool.mount
```

**Step 4 — Verify both pools are mounted:**
```bash
df -h | grep mergerfs
# Should show two entries: /srv/mergerfs/data and /srv/mergerfs/pool
ls /srv/mergerfs/pool
```

---

### 10.3 Set Up the Nightly Cache Flush

The NVMe is a **write buffer**. Every night, everything on it is moved to Pool 1, where `epmfs` distributes the incoming files across whichever HDD has the most free space. The key: the mover dumps to `/srv/mergerfs/data` (Pool 1), not to a specific drive UUID. MergerFS handles the distribution automatically.

After the flush, SnapRAID syncs against the individual HDD drives and parity covers everything.

**Find your NVMe cache UUID path** (you already noted this above):
```bash
lsblk -f
# NVMe cache partition mounts at /srv/dev-disk-by-uuid-<nvme-cache-uuid>
```

**Create the scheduled task:**

`System` → `Scheduled Tasks` → **+**

| Setting | Value |
|---------|-------|
| Enable | Yes |
| Time | `0 3 * * *` (3:00 AM every night) |
| Command | See below |

Command (substitute your NVMe UUID path):
```bash
#!/bin/bash
set -euo pipefail

NVME="/srv/dev-disk-by-uuid-<nvme-cache>"
POOL1="/srv/mergerfs/data"

# Move all files from NVMe to Pool 1, excluding SnapRAID content files.
# rsync --remove-source-files is used instead of mv: it copies first, then
# deletes the source only on success. This makes the operation resumable and
# safe across a power loss — a partial file on the destination will not cause
# data loss, and the original on the NVMe remains intact until the copy succeeds.
rsync -a --remove-source-files \
  --exclude='snapraid*' \
  "$NVME/" "$POOL1/"

# Clean up any empty directories left behind on the NVMe
find "$NVME/" -mindepth 1 -empty -type d -delete
```

> `set -euo pipefail` causes the script to abort immediately on any error rather than continuing silently. If rsync fails mid-transfer (e.g. power loss, full disk), the next run will resume from where it left off — rsync skips files that already exist at the destination. *Search: rsync remove-source-files resumable*

**How the full nightly sequence works:**
1. **3:00 AM** — Mover empties NVMe into Pool 1. `epmfs` spreads files across HDDs.
2. **3:30 AM** — SnapRAID syncs against individual HDD drives. Parity now covers everything that just arrived.
3. **Morning** — NVMe is empty and ready for a new day. All data is on HDDs under parity protection.

> **Why move before sync?** SnapRAID only sees the HDDs. If sync ran first, files still on the NVMe would be missing from parity until the following night — unprotected. Move first, sync after, always.

---

## Phase 11 — Create Shared Folders

Shared folders are logical divisions within your MergerFS pool. Everything lives inside `/srv/mergerfs/pool`.

`Storage` → `Shared Folders` → click **+**

Create the following folders (adjust to your needs):

| Name | File System | Relative Path | Purpose |
|------|-------------|---------------|---------|
| `documents` | pool (mergerfs) | `documents/` | Documents & files |
| `backups` | pool (mergerfs) | `backups/` | PC/device backups |
| `photos` | pool (mergerfs) | `photos/` | Photo library |
| `media` | pool (mergerfs) | `media/` | Optional media files |

For each, select the **MergerFS pool** filesystem and set appropriate permissions.

---

## Phase 12 — Configure SMB (Windows File Sharing)

### 12.1 Enable SMB Service

`Services` → `SMB/CIFS` → `Settings` → Enable → set Workgroup to match your network (default `WORKGROUP` is fine) → Save → Apply.

### 12.2 Create a NAS User

`Users` → `Users` → **+**  
Create a user (e.g., your own username). This is the account you'll use to connect from Windows/Mac.

### 12.3 Add SMB Shares

`Services` → `SMB/CIFS` → `Shares` → **+**

For each shared folder you want to expose, create a share:

| Setting | Value |
|---------|-------|
| Shared folder | Select from your list (e.g., `documents`) |
| Comment | Optional description |
| Public | No (require login) |
| Browseable | Yes |
| Inherit permissions | Yes |
| Hosts allow | Optionally restrict to your LAN, e.g., `192.168.1.0/24` |

Repeat for each share. Save → Apply.

### 12.4 Set Folder Permissions

`Storage` → `Shared Folders` → select a folder → `Permissions`  
Grant your user **Read/Write** access to each folder.

### 12.5 Connect from Windows

Open File Explorer → address bar → type:
```
\\<nas-ip>
```
Enter your NAS username and password when prompted.

---

## Phase 13 — Configure NFS (Linux / VM Sharing)

### 13.1 Enable NFS Service

`Services` → `NFS` → `Settings` → Enable → Save → Apply.

### 13.2 Add NFS Shares

`Services` → `NFS` → `Shares` → **+**

| Setting | Value |
|---------|-------|
| Shared folder | Select the folder (e.g., `photos`) |
| Client | Your LAN CIDR, e.g., `192.168.1.0/24` |
| Privilege | Read/Write |
| Squash | `No root squash` (or `All squash` for stricter security) |
| Extra options | `async,no_subtree_check` |

Save → Apply.

### 13.3 Mount from a Linux Client

```bash
sudo mount -t nfs <nas-ip>:/export/<share-name> /mnt/nas-share
```

To make it permanent, add to `/etc/fstab`:
```
<nas-ip>:/export/photos  /mnt/photos  nfs  defaults,_netdev  0  0
```

---

## Phase 14 — Configure SSH (Nice to Have)

SSH is already enabled from Phase 4. A few hardening steps are recommended:

### 14.1 Create a Non-Root User for SSH (Optional but Recommended)

If you created a user in Phase 12, you can already SSH as that user. To also allow root SSH (convenient for admin tasks, less secure):

```bash
# Already enabled since you installed OMV — root SSH is on by default
# To verify:
grep PermitRootLogin /etc/ssh/sshd_config
```

### 14.2 (Optional) Key-Based Authentication

On your client PC, generate a key pair if you don't have one:
```bash
ssh-keygen -t ed25519
```

Copy the key to the NAS:
```bash
ssh-copy-id root@<nas-ip>
```

After confirming key login works, you can disable password SSH:
```bash
# Edit on the NAS:
nano /etc/ssh/sshd_config
# Set: PasswordAuthentication no
systemctl restart ssh
```

---

## Phase 15 — Scheduled Maintenance

### Why you must never run `snapraid sync` blindly

This is the most important thing to understand about automating SnapRAID.

`snapraid sync` is not safe to run unconditionally. It does exactly what it's told: it looks at the current state of your drives and updates parity to match. It has no concept of "something is wrong" — if a drive died overnight and all its files appear missing, sync will faithfully update parity to reflect those missing files, **permanently destroying your ability to recover them**.

The correct pattern is always: **diff first, sync only if diff looks normal.**

`snapraid diff` counts how many files have been added, deleted, modified, or moved since the last sync. On a normal night this is small numbers — maybe 50 new photos added, 0 deleted. If a drive dies, diff suddenly shows tens of thousands of files as deleted. That's your signal to stop, not to proceed.

The way this is enforced is with a **delete threshold**: if the number of deleted files exceeds a configured number, the sync is aborted and you are notified. This single check turns a dangerous automated sync into a safe one.

> **Future-you reading this after something went wrong:** If the nightly job stopped and sent you a warning about too many deleted files — **do not force a sync**. That warning just saved your data. Check your drives with `snapraid status` and `lsblk -f` first. If a drive is missing from the output, go to the Drive Failure Recovery section. Only run a manual sync once you understand why the deletion count spiked.

---

### 15.1 Set Up the Mount Guard Script

The diff script's delete threshold catches a dead drive *after* SnapRAID reads the drives and sees files missing. But there's an even earlier safety check: verify that every drive is actually mounted before SnapRAID runs at all. If a drive died and its mount point is simply absent, we should abort immediately — before SnapRAID even opens a file.

This is set up as a small wrapper script. Instead of calling the diff script directly from OMV's scheduler, the scheduler calls this wrapper, which checks mounts first and then hands off to the diff script. One scheduled task, two layers of protection.

**Create the wrapper script** via SSH:

```bash
nano /usr/local/bin/snapraid-safe-sync.sh
```

Paste this content (substitute your actual UUID paths):

```bash
#!/bin/bash
# snapraid-safe-sync.sh
# Checks all SnapRAID drives are mounted before allowing diff/sync to run.
# If any drive is missing, aborts and exits with error (OMV will email on non-zero exit).

DRIVES=(
  "/srv/dev-disk-by-uuid-<data-drive-uuid>"
  "/srv/dev-disk-by-uuid-<parity-drive-uuid>"
)

for DRIVE in "${DRIVES[@]}"; do
  if ! grep -q "$DRIVE" /proc/mounts; then
    echo "ERROR: Drive not mounted: $DRIVE"
    echo "SnapRAID sync aborted to prevent data loss."
    echo "Check your drives immediately. Do NOT run snapraid sync manually until you know why."
    exit 1
  fi
done

echo "All drives confirmed mounted. Proceeding with SnapRAID diff script."

# Verify the diff script path before relying on this script:
# which omv-snapraid-diff
# OMV plugin updates have occasionally changed this path. If the command
# below fails, run 'which omv-snapraid-diff' and update the path here.
/usr/sbin/omv-snapraid-diff
```

Make it executable:
```bash
chmod +x /usr/local/bin/snapraid-safe-sync.sh
```

> When you add drives in the future, add their UUID mount paths to the `DRIVES` array in this script. One new line per drive. *Search: bash array append*

> If you later switch to the AIO script (see section 15.2), replace `/usr/sbin/omv-snapraid-diff` with `/usr/local/bin/snapraid-aio-script.sh` — the mount check wrapper works the same either way.

**Schedule the wrapper** (not the diff script directly):

`System` → `Scheduled Tasks` → **+**

| Setting | Value |
|---------|-------|
| Enable | Yes |
| Time | `30 3 * * *` (3:30 AM every night) |
| Command | `/usr/local/bin/snapraid-safe-sync.sh` |
| Send email on error | Yes |

With "Send email on error" enabled in OMV, a non-zero exit from the script (triggered by a missing drive) will fire an email notification automatically — no extra mail configuration needed in the script itself.

**What happens now if a drive dies overnight:**

1. At 3:00 AM — cache flush runs normally
2. At 3:30 AM — mount guard checks all drive UUIDs against `/proc/mounts`
3. Dead drive's mount point is absent → script exits with error
4. OMV emails you: "snapraid-safe-sync.sh failed"
5. Diff script never runs, SnapRAID never runs, parity is untouched
6. You wake up, read the email, go to the Drive Failure Recovery section

---

### 15.2 Configure SnapRAID Diff Script (thresholds)

The OMV SnapRAID plugin includes a built-in diff script already installed alongside the plugin. It runs: diff → checks thresholds → syncs if safe → scrubs on schedule → emails results. The mount guard (above) calls this automatically.

> **Alternatively, use the AIO script** (section 15.3) instead of the OMV diff script — it has smarter threshold logic, richer email reports, and SMART status. The choice only affects which tool the mount guard hands off to. For most users the OMV diff script is sufficient to start with.

**Set up the diff script:**

`Services` → `SnapRAID` → `Settings`

| Setting | Value | Why |
|---------|-------|-----|
| **Delete threshold** | `500` | Abort sync if more than 500 files appear deleted. On a media NAS, batch deletes of a TV season, duplicates, or reorganised folders can easily be 50–300 files — a threshold of 20 would produce constant false alarms for normal usage. 500 is high enough to survive routine large deletes but low enough that a dead drive (showing thousands or millions of deletions) still aborts cleanly. Tune this up or down based on your own deletion patterns. |
| **Update threshold** | `40` | Abort if more than 40 files appear modified unexpectedly. Protects against mass corruption being synced into parity. |
| **Run scrub** | Yes | Runs scrub automatically after a successful sync. |
| **Scrub percentage** | `22` | Checks 22% of the array per run — full verification over ~4 Sundays. |
| **Scrub frequency** | `7` | Only scrub files that haven't been scrubbed in the last 7 days — avoids redundant work. |
| **Pre-hash** | Yes (optional) | Reads each file twice before computing parity — catches corruption during the read itself. Slower but safer. *Search: snapraid pre-hash* |

Save. Do **not** click "Schedule Diff" — the mount guard script already calls the diff script. Scheduling it separately would run it twice.

> **Future-you reading this after something went wrong:** If the nightly job stopped and sent you a warning about too many deleted files — **do not force a sync**. That warning just saved your data. Check your drives with `snapraid status` and `lsblk -f` first. If a drive is missing from the output, go to the Drive Failure Recovery section. Only run a manual sync once you understand why the deletion count spiked.

**The full nightly sequence:**

1. **3:00 AM** — Cache flush moves everything from NVMe to Pool 1 (HDD)
2. **3:30 AM** — Mount guard checks all drives are present
   - Any drive missing → abort, email, stop. SnapRAID never runs.
   - All drives present → hand off to diff script
3. **3:30 AM** (continued) — Diff script runs:
   - Counts added/deleted/modified files since last sync
   - Deletions > 500 or updates > 40 → **abort, send warning, do not sync**
   - Within thresholds → run sync, then scrub 22% of the array
4. **Morning** — Clean "sync completed" email, or a warning that needs your attention

---

### 15.3 (Optional) Upgrade to snapraid-aio-script

The OMV diff script is sufficient for most users. If you want more control — smarter threshold logic, richer email reports, SMART status included in every notification — the community's preferred tool is **snapraid-aio-script** by auanasgheps. It is explicitly tested on OMV 6 and OMV 7.

It works identically in concept (diff → threshold → sync → scrub) but adds:
- `ADD_DEL_THRESHOLD` — allows a sync that would breach the delete threshold *if additions far outnumber deletions*, useful after a big file reorganisation
- "Sync after N consecutive warnings" — eventually forces through after repeated false alarms you've acknowledged
- SMART drive health report included in every email

Install:
```bash
wget -O /usr/local/bin/snapraid-aio-script.sh \
  https://raw.githubusercontent.com/auanasgheps/snapraid-aio-script/master/snapraid-aio-script.sh
chmod +x /usr/local/bin/snapraid-aio-script.sh

wget -O /etc/snapraid-aio-script.conf \
  https://raw.githubusercontent.com/auanasgheps/snapraid-aio-script/master/script-config.sh
nano /etc/snapraid-aio-script.conf
```

Key settings:
```bash
DEL_THRESHOLD=500
UP_THRESHOLD=40
SCRUB_PERCENT=22
SCRUB_DELAYED_RUN=7
EMAIL_ADDRESS="you@example.com"
```

Then update the mount guard script — replace `/usr/sbin/omv-snapraid-diff` with `/usr/local/bin/snapraid-aio-script.sh`. The scheduled task in OMV stays unchanged.

> **Note:** The AIO script requires at least one prior sync to have completed — the initial sync from Phase 8 satisfies this.

### 15.4 (Optional) Btrfs Scheduled Scrub

Btrfs has its own independent scrub that verifies filesystem-level checksums, separate from SnapRAID. Only the data drive and NVMe are Btrfs — the parity drive is now ext4 and does not need a Btrfs scrub.

`System` → `Scheduled Tasks` → **+**

| Setting | Value |
|---------|-------|
| Enable | Yes |
| Time | `0 12 1 * *` (12:00 noon on the 1st of each month) |
| Command | See below |

```bash
btrfs scrub start /srv/dev-disk-by-uuid-<data-drive-uuid>
btrfs scrub start /srv/dev-disk-by-uuid-<nvme-cache-uuid>
```

> Scheduled at noon rather than early morning to avoid I/O contention with the nightly SnapRAID sync which runs at 3:30 AM. On the 1st of the month, running both within 90 minutes of each other on an 18 TB drive would cause significant contention.

---

## Phase 16 — (Optional) HDD Spindown with hd-idle

> **Reference:** The OMV community maintains an up-to-date guide for this at `https://forum.openmediavault.org/index.php?thread/37438-how-to-spin-down-hard-drives-with-hd-idle/` — worth checking if anything below doesn't match what you see, as package names and repo URLs can shift between hd-idle versions.

With the NVMe cache absorbing all writes during the day, your HDDs genuinely sit idle for long stretches — they only wake at 3 AM for the cache flush and sync. Without spindown they spin continuously at ~5–8W each doing nothing. hd-idle puts them to sleep when idle and wakes them transparently on access.

**Why not use OMV's built-in spindown?** OMV uses `hdparm` for spindown, which is unreliable — drives frequently fail to stay spun down, or the setting doesn't apply consistently to all drive models. The community consensus is to disable OMV's built-in spindown entirely and use hd-idle instead.

**Why the adelolmo version specifically?** The `hd-idle` package in the standard Debian repository is old and buggy. Adelolmo rewrote it and maintains an updated version that works correctly on modern drives and OMV 7.

---

### 16.1 Disable OMV's Built-in Spindown

Do this first to avoid conflicts between hdparm and hd-idle.

`Storage` → `Disks` → edit each HDD (not the NVMe) and set:

| Setting | Value |
|---------|-------|
| Advanced Power Management | `Disabled` (or a value between 128–255 if the field requires a number — lower values interfere with hd-idle) |
| Spindown time | `Disabled` |
| Write cache | `Enabled` (leave this on) |

Save and apply for each drive.

---

### 16.2 Install hd-idle

Add adelolmo's repository — this ensures you get the correct version and it stays updated with `apt upgrade`:

```bash
apt install apt-transport-https -y

curl -fsSLo /usr/share/keyrings/adelolmo-archive-keyring.gpg \
  https://adelolmo.github.io/andoni.delolmo@gmail.com.gpg

echo "deb [signed-by=/usr/share/keyrings/adelolmo-archive-keyring.gpg] \
  https://adelolmo.github.io/$(lsb_release -cs) $(lsb_release -cs) main" \
  | tee /etc/apt/sources.list.d/adelolmo.github.io.list

apt update && apt install hd-idle -y
```

---

### 16.3 Configure hd-idle

The configuration file is `/etc/default/hd-idle`. Open it:

```bash
nano /etc/default/hd-idle
```

Replace the contents with this (substitute your actual drive labels):

```bash
START_HD_IDLE=true

HD_IDLE_OPTS="-i 0 \
  -a /dev/disk/by-label/data1   -i 1800 \
  -a /dev/disk/by-label/parity1 -i 1800 \
  -l /var/log/hd-idle.log"
```

**What each part does:**

- `START_HD_IDLE=true` — enables the daemon at boot
- `-i 0` at the start — disables spindown for all drives **not** explicitly listed below. This is critical: it prevents hd-idle from accidentally spinning down the NVMe or any other drive you didn't intend. *Search: hd-idle -i 0 default*
- `-a /dev/disk/by-label/data1 -i 1800` — spin down the data drive after 1800 seconds (30 minutes) of inactivity. Uses the disk label rather than `/dev/sda` — labels are stable across reboots; device names are not. *Search: hd-idle by-label*
- `-a /dev/disk/by-label/parity1 -i 1800` — same for the parity drive. The parity drive is particularly idle — it only wakes for the nightly sync — so 30 minutes is generous
- `-l /var/log/hd-idle.log` — write spindown/wakeup events to a log file so you can verify it's working

> **When you add drives:** add a new `-a /dev/disk/by-label/<label> -i 1800` line for each new HDD. The NVMe never needs an entry — SSDs don't spin.

> **30 minutes** is a reasonable starting point. The drives wake at 3 AM for the cache flush and sync, spin for however long that takes (~15–60 minutes depending on data volume), then spin back down. During the day they only wake when you actually access a file.

---

### 16.4 Enable and Start hd-idle

```bash
systemctl enable hd-idle
systemctl start hd-idle
```

Verify it's running:
```bash
systemctl status hd-idle
```

---

### 16.5 Set Up Log Rotation

The hd-idle log will grow indefinitely without rotation. Create a logrotate config:

```bash
nano /etc/logrotate.d/hd-idle
```

Contents:
```
/var/log/hd-idle.log {
    weekly
    rotate 4
    compress
    missingok
    notifempty
}
```

---

### 16.6 Verify Spindown Is Working

After 30+ minutes of no disk activity, check the log:
```bash
tail /var/log/hd-idle.log
```

You should see entries like:
```
2026-01-15T03:47:22 sda spindown
2026-01-15T03:47:22 sdb spindown
```

To force an immediate test without waiting:
```bash
# Spin down a specific drive right now
hd-idle -t /dev/disk/by-label/data1
```

Then check `dmesg` or the log to confirm. You can also verify a drive is spun down with:
```bash
smartctl -n standby /dev/sda
# Returns "STANDBY" if spun down, or reads SMART data if spinning
```

---

### 16.7 AIO Script Integration (if using 15.3)

If you use the AIO script instead of the OMV diff script, it can automatically spin drives down after the nightly sync completes. In `/etc/snapraid-aio-script.conf`:

```bash
SPINDOWN=1
```

The AIO script calls hd-idle internally after sync finishes, so your drives go back to sleep promptly after the 3:30 AM job rather than waiting for the 30-minute idle timer to expire.

---

## Phase 17 — Essential Best Practices

Four additions that complete the setup. Each is independent — do them in any order.

---

### 17.1 Email Notifications (Do This First)

Every safety net in this guide — the mount guard, the diff threshold, SMART alerts, scheduled task failures — sends you an email when something goes wrong. Without email configured, all of those protections are silent.

> **This section uses Gmail.** The settings are Gmail-specific, but the OMV notification system works with any SMTP provider. For other providers substitute your own SMTP server, port, and credentials. Common alternatives: Outlook/Hotmail (`smtp.office365.com`, port 587), Fastmail (`smtp.fastmail.com`, port 587), or a self-hosted relay.

---

**Step 1 — Create a Gmail App Password**

Google requires an App Password rather than your regular Gmail password for SMTP access. Your normal password will not work.

1. Go to `https://myaccount.google.com/security`
2. Ensure **2-Step Verification** is enabled (required for App Passwords)
3. Search for "App passwords" in the search bar on your Google account page
4. Click **App passwords**, name it something like `NAS`, and click **Create**
5. Google generates a **16-character password** — copy it immediately, you won't see it again

> **Tip:** You can use a Gmail alias as the sender address — e.g. `yourname+nas@gmail.com` instead of `yourname@gmail.com`. Gmail ignores the `+nas` part and delivers to the same inbox, but it makes NAS notifications easy to filter.

---

**Step 2 — Configure OMV Notifications**

`System` → `Notifications` → `Settings`

Enter the following values:

| Field | Value |
|-------|-------|
| Enable | Yes |
| SMTP server | `smtp.gmail.com` |
| SMTP port | `587` |
| Encryption | `STARTTLS` |
| Sender email | `yourname@gmail.com` (or your `+nas` alias) |
| Authentication | Yes |
| Username | `yourname@gmail.com` |
| Password | The 16-character App Password from Step 1 |
| Primary recipient | `yourname@gmail.com` (where you want to receive alerts) |

Click **Save**, then click **Test** — you should receive a test email within a minute. Check spam if it doesn't arrive.

> **If the test shows success but no email arrives:** check that your router's DNS is configured (not blank) in OMV's network settings — postfix needs DNS to resolve `smtp.gmail.com`. `System` → `Network` → `General` → DNS server: add `8.8.8.8` if empty.

> **Troubleshooting:** check `/var/log/mail.log` for postfix errors. The most common cause of failure is an incorrect or expired App Password — generate a fresh one and try again.

---

**Step 3 — Set Your User Email Address**

OMV routes scheduled task output to the user that runs the task. Make sure your admin user has an email address set so nothing gets silently dropped:

`Users` → `Users` → edit your admin user → set **Email** to your Gmail address → Save.

---

### 17.2 Scheduled S.M.A.R.T. Tests

Phase 6 covered manually checking SMART at setup. Scheduling weekly automated short tests means you get early warning of a degrading drive months before it fails — drives rarely die suddenly; they usually show symptoms first.

`Services` → `S.M.A.R.T.` → `Scheduled Tests` → **+**

Create one entry for each HDD (not the NVMe — SSDs report different attributes):

| Setting | Value |
|---------|-------|
| Drive | Select your data drive (Bay 1) |
| Test type | `Short` |
| Schedule | `0 1 * * 0` (1:00 AM every Sunday) |

Repeat for the parity drive (Bay 2). Two entries total to start, add more when you expand.

SMART will email you if a drive reports a failure — since you've now configured notifications in 17.1, these alerts will reach you automatically.

---

### 17.3 `snapraid touch` Before Sync

SnapRAID uses sub-second file timestamps to tell the difference between a moved file and a delete+add pair. Some tools (certain copy utilities, some apps) create files with a zero sub-second timestamp, which tricks SnapRAID into treating moves as deletions — which inflates your deletion count and can falsely trip the diff threshold.

`snapraid touch` stamps those files before sync runs, fixing this silently.

**If you're using the AIO script (Phase 15.3):** check that `TOUCH=1` is set in your `/etc/snapraid-aio-script.conf` — it is not always the default depending on the version you downloaded. Open the config and confirm:
```bash
grep TOUCH /etc/snapraid-aio-script.conf
# Should show: TOUCH=1
# If it shows TOUCH=0, change it to TOUCH=1 and save
```
If TOUCH=1 is set, touch runs automatically and you don't need the scheduled task below.

**If you're using the OMV diff script (Phase 15.2):** add `snapraid touch` as a scheduled task that runs just before the mount guard:

`System` → `Scheduled Tasks` → **+**

| Setting | Value |
|---------|-------|
| Enable | Yes |
| Time | `25 3 * * *` (3:25 AM — 5 minutes before the mount guard at 3:30) |
| Command | `snapraid touch` |

Five minutes is plenty — touch is fast. The sequence is now: cache flush (3:00) → touch (3:25) → mount guard → diff → sync (3:30).

---

### 17.4 UPS — Power Loss Protection

A NAS writing to spinning HDDs with no battery backup is one power cut away from filesystem corruption. MergerFS and Btrfs handle unexpected shutdowns reasonably well, but it is a real risk — especially if power cuts out mid-sync while SnapRAID is writing parity.

Even a basic consumer UPS (APC, Eaton, CyberPower) with a USB monitoring cable will gracefully shut the NAS down when the battery runs low, rather than letting it die mid-write.

OMV supports UPS monitoring natively via the `nut` package:

`Services` → `UPS` → Enable → connect your UPS via USB → OMV will detect it and show battery status. Set the shutdown threshold (e.g. shut down when battery is below 50% or runtime below 5 minutes).

> For a NAS of this size (two 18TB drives spinning), a UPS rated at 600–1000VA provides 5–15 minutes of runtime — enough to cleanly shut down even mid-sync.

This is optional but strongly recommended if your area has occasional power cuts, or if your NAS is storing photos that are genuinely irreplaceable.

---

### 17.5 3-2-1 Backup Reminder

**SnapRAID is not a backup.** It protects against drive failure and bitrot. It does not protect against:

- Accidental deletion that has already been synced (the deleted files are now gone from parity too)
- Ransomware that encrypts your files and triggers a sync
- Theft or fire taking out the whole NAS
- A bug or misconfiguration destroying the array

The industry standard is **3-2-1**: 3 copies of your data, on 2 different media types, with 1 copy offsite.

For a home NAS storing photos, a practical approach:

- **Local copy** — your NAS (Pool 1, under SnapRAID protection)
- **External drive backup** — a USB drive you periodically connect and run `rsync` to, then store somewhere else in the house
- **Offsite/cloud** — `rclone` syncing your photos folder to Backblaze B2 (~$6/TB/month), or a cloud storage provider of your choice

The external drive and rclone steps are outside the scope of this guide, but worth setting up once the NAS is running. *Search: rclone backblaze b2 setup*, *Search: rsync external drive backup linux*

---

## Phase 18 — Verify Everything Is Working

Run through this checklist before declaring the NAS done:

- [ ] OMV web UI accessible at `http://<static-ip>`
- [ ] SSH works: `ssh root@<nas-ip>` or `ssh <user>@<nas-ip>`
- [ ] Both MergerFS pools mounted: `df -h | grep mergerfs` shows both `/srv/mergerfs/data` and `/srv/mergerfs/pool`
- [ ] Pool 2 systemd unit enabled: `systemctl is-enabled srv-mergerfs-pool.mount`
- [ ] SnapRAID sync completes without errors: `snapraid status`
- [ ] SMB shares visible from Windows: `\\<nas-ip>\`
- [ ] NFS shares mountable from Linux client
- [ ] Write a test file to the SMB share, verify it appears in the pool, verify a sync picks it up
- [ ] Email notifications working: `System` → `Notifications` → Test button sends successfully
- [ ] SMART scheduled tests configured for both HDDs
- [ ] hd-idle running and logging: `systemctl status hd-idle` and `tail /var/log/hd-idle.log`
- [ ] Btrfs filesystems healthy: `btrfs filesystem show` and `btrfs device stats /dev/sda`

---

## Phase 19 — Immich on Your Home Server (Storage on NAS)

Immich runs on your **home server** via Docker. Its library — all photos and videos — lives on the **NAS**. The home server mounts the NAS share over NFS and Immich writes there directly.

```
[Phone / PC]  →  uploads to  →  [Immich on Home Server]
                                         ↓ (SMB/CIFS mount)
                                 [NAS: /srv/mergerfs/pool/photos]
```

### 19.1 Prepare the NAS Side

The `photos` SMB share from Phase 12 is all you need. No NFS setup required for Immich.

The one thing worth double-checking: the NAS user you created has **read/write access** to the `photos` shared folder.

`Storage` → `Shared Folders` → `photos` → `Permissions` → confirm your NAS user has Read/Write.

### 19.2 Mount the NAS Share on the Home Server

Immich will access the NAS via an SMB share mounted through `/etc/fstab`. The key extra option is `nobrl` — this disables byte-range locking, which otherwise causes conflicts between Docker and CIFS mounts.

First, install the CIFS utilities if not already present:
```bash
sudo apt install cifs-utils
```

Create a credentials file (keep your password out of fstab):
```bash
sudo nano /etc/nas-credentials
```
Contents:
```
username=<your-nas-user>
password=<your-nas-password>
```
Lock it down:
```bash
sudo chmod 600 /etc/nas-credentials
```

Create the mount point and test:
```bash
sudo mkdir -p /mnt/nas/photos

# Test mount first:
sudo mount -t cifs //<nas-ip>/photos /mnt/nas/photos \
  -o credentials=/etc/nas-credentials,uid=1000,gid=1000,nobrl

# Verify read/write works:
ls /mnt/nas/photos
touch /mnt/nas/photos/test.txt && rm /mnt/nas/photos/test.txt
```

Once confirmed, make it **permanent** in `/etc/fstab`:
```
//<nas-ip>/photos  /mnt/nas/photos  cifs  credentials=/etc/nas-credentials,uid=1000,gid=1000,file_mode=0770,dir_mode=0770,nobrl,_netdev  0  0
```

> `uid=1000,gid=1000` should match the user that runs Docker on your home server. Check with `id $(whoami)`.

Reload:
```bash
sudo systemctl daemon-reload
sudo mount -a
```

### 19.3 Install Immich on the Home Server

Immich uses Docker Compose. On your home server:

```bash
# Create Immich directory
mkdir -p ~/immich && cd ~/immich

# Download the official compose file and env template
wget -O compose.yaml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
```

### 19.4 Configure Immich to Use NAS Storage

Edit the `.env` file:
```bash
nano .env
```

Set these values:

```env
# Where Immich stores uploaded photos — point this to your NAS mount
UPLOAD_LOCATION=/mnt/nas/photos

# Where Immich stores its database (keep this LOCAL on the home server — not on NAS)
DB_DATA_LOCATION=./postgres

# Timezone
TZ=Europe/Berlin
```

> **Important**: Keep `DB_DATA_LOCATION` on local storage (the home server's own disk). The Immich database (PostgreSQL) does not perform well over NFS. Only the photo library (`UPLOAD_LOCATION`) goes to the NAS.

### 19.5 Start Immich

```bash
docker compose up -d
```

Immich will start and be accessible at:
```
http://<home-server-ip>:2283
```

First launch creates the admin account — follow the setup wizard.

### 19.6 Point Immich at Existing Photos (External Library)

If you already have photos on the NAS that you want Immich to index without moving them, use Immich's **External Library** feature:

In Immich web UI:
1. `Administration` → `Libraries` → `Create Library` → `External`
2. Set the import path to the folder inside your NAS mount, e.g., `/mnt/nas/photos/existing`
3. Run a scan — Immich indexes the files without copying them

This way, photos you manage manually (e.g., organized folders from a camera) stay in place and are still visible in Immich.

### 19.7 Keep Immich Updated

Immich releases frequently. Update with:
```bash
cd ~/immich
docker compose pull
docker compose up -d
```

> **Tip**: Immich is under active development — check the release notes before updating, as breaking changes occasionally occur between versions.

### 19.8 Backup the Immich Database

The photos themselves are on the NAS (and protected by SnapRAID), but the Immich database (albums, faces, metadata) lives on the home server. Back it up separately:

```bash
# Add to home server crontab (daily at 1 AM):
0 1 * * * docker exec immich_postgres pg_dumpall -U postgres > /mnt/nas/backups/immich-db-$(date +\%F).sql
```

This dumps the database to the NAS `backups` share, so it's also protected by SnapRAID.

> **Circular dependency warning:** if the NAS is unreachable when this runs (network issue, NVMe failure, HDD failure), the backup silently fails — writing to a dead mount just errors out. Add a local fallback so you always have at least one copy regardless of NAS availability:
> ```bash
> 0 1 * * * docker exec immich_postgres pg_dumpall -U postgres \
>   > ~/immich-db-backup/immich-db-$(date +\%F).sql && \
>   cp ~/immich-db-backup/immich-db-$(date +\%F).sql \
>   /mnt/nas/backups/immich-db-$(date +\%F).sql
> ```
> This writes locally first, then copies to the NAS. If the NAS copy fails, the local copy remains. Keep only the last 7 days locally to avoid filling the home server disk (`find ~/immich-db-backup -mtime +7 -delete`).

---

## Useful Commands Reference

```bash
# Check overall disk status
lsblk -f

# Check MergerFS pool usage (both pools)
df -h /srv/mergerfs/data
df -h /srv/mergerfs/pool

# Check Pool 2 systemd unit status
systemctl status srv-mergerfs-pool.mount

# Check SnapRAID status
snapraid status

# Run SnapRAID sync manually (only in recovery/expansion situations — see Phase 15 for why)
snapraid sync

# Run SnapRAID diff (always run before any manual sync to check for unexpected deletions)
snapraid diff

# Run snapraid touch (stamps zero-timestamp files — run manually before diff if needed)
snapraid touch

# Run SnapRAID scrub manually
snapraid scrub

# Check Btrfs filesystem health
btrfs filesystem show
btrfs device stats /dev/sda
btrfs device stats /dev/sdb

# Check mounted filesystems
mount | grep btrfs

# Check hd-idle status
systemctl status hd-idle

# Check hd-idle spindown log
tail /var/log/hd-idle.log

# Force immediate spindown of a drive (for testing)
hd-idle -t /dev/disk/by-label/data1

# Check if a drive is currently spun down
smartctl -n standby /dev/sda

# Check NFS service status
systemctl status nfs-server

# Restart OMV services after config change
omv-salt deploy run all
```

---

---

## Drive Failure Recovery

Three different drives can fail, and each is handled differently. Read the relevant section calmly when the time comes — none of these are emergencies if you act methodically.

---

### Scenario A — NVMe Cache Dies

**What you lose:** Everything written to the NVMe since the last nightly flush (at most ~24 hours of new writes). The content file copy on the NVMe is also gone, but you still have copies on both HDDs.

**Your data on the HDDs is completely unaffected.** SnapRAID never touched the NVMe.

**Steps:**

1. **Don't panic.** Pool 2 will fail to mount on next boot, but Pool 1 (`/srv/mergerfs/data`) and your HDD data are intact.

2. **Stop Pool 2:**
   ```bash
   systemctl stop srv-mergerfs-pool.mount
   ```

3. **Temporarily point shares at Pool 1** so you retain access to your files while the NVMe is being replaced:
   In OMV, edit your shared folders to use the `data` pool (`/srv/mergerfs/data`) instead of `pool`. Your data is all there.

4. **Replace the NVMe.** Any NVMe of the same size or larger works.

5. **Re-partition and format** the new NVMe exactly as in Phases 3 and 7:
   - Partition 1: ~60 GB ext4 for OS — not needed if you're only replacing the cache partition
   - Partition 2: remaining space, Btrfs, label `cache`
   > If the full NVMe died including the OS partition, reinstall OMV from scratch following Phases 1–5, then resume from step 6 below.

6. **Mount the new cache partition** in OMV (`Storage` → `File Systems`).

7. **Restore Pool 2** — edit `/etc/systemd/system/srv-mergerfs-pool.mount` with the new NVMe UUID, then:
   ```bash
   systemctl daemon-reload
   systemctl start srv-mergerfs-pool.mount
   ```

8. **Update shared folders** back to the `pool` mount point.

9. **No SnapRAID action needed** — the HDDs are unchanged, parity is intact. Just let the next nightly sync run and it will recreate the content file on the NVMe.

---

### Scenario B — HDD Data Drive Dies (Bay 1, 3, or 4)

**What you lose:** The files physically stored on that specific drive. Files on other drives in Pool 1 are unaffected.

**SnapRAID can reconstruct the lost files** from parity, as long as you run the recovery before replacing the drive and writing new data.

**Steps:**

1. **Do not run `snapraid sync`** before recovering. A sync after a drive failure would update parity to reflect the missing files — permanently losing the ability to recover them. *Search: snapraid fix before sync*

2. **Identify which files were on the failed drive:**
   ```bash
   snapraid status
   ```
   This will show the failed drive and list affected files.

3. **Install a replacement drive.** Format it with Btrfs and give it the same label as the failed drive:
   ```bash
   mkfs.btrfs /dev/sdX -L "data1"   # use the same label as the dead drive
   ```

4. **Mount the new drive in OMV** (`Storage` → `File Systems`). Because the label matches, SnapRAID and MergerFS configs don't need to change.

5. **Run SnapRAID fix** to reconstruct the lost files onto the new drive:
   ```bash
   snapraid fix -d <drive-name>
   # <drive-name> is what you called this disk in SnapRAID config, e.g. "d1"
   ```
   This reads parity from Bay 2 and reconstructs everything. On 18 TB this takes many hours — let it run.

6. **Verify the recovery:**
   ```bash
   snapraid check -d <drive-name>
   ```
   Should report no errors.

7. **Run a manual sync** to recompute parity now that the array is healthy again:
   ```bash
   snapraid sync
   ```
   > This is a deliberate, intentional sync after a known-good recovery — running it directly is correct here. The diff script's threshold check is not needed because you are the one who initiated this, you know exactly why files were added/changed (they were just reconstructed by `fix`), and the array is in a known good state.

8. **No MergerFS changes needed.** Because the drive has the same label and UUID-based mount path, Pool 1 sees it as the same drive returning.

---

### Scenario C — HDD Parity Drive Dies (Bay 2)

**What you lose:** Nothing — the parity drive contains no actual data, only the parity calculations and a content file copy. Your files are all safe.

**What you lose temporarily:** Protection. Until you replace the parity drive and resync, a second drive failure would be unrecoverable. Replace it promptly.

**Steps:**

1. **Verify your data is intact** — mount Pool 1 and check your files are accessible. They should be completely unaffected.

2. **Replace the parity drive** with a new drive of the same size or larger. Format with ext4:
   ```bash
   mkfs.ext4 -L "parity1" /dev/sdX
   ```

3. **Mount the new drive in OMV** (`Storage` → `File Systems`).

4. **Run a manual sync** to recompute parity onto the new drive:
   ```bash
   snapraid sync
   ```
   This will take many hours on 18 TB — it is computing full parity from scratch across all data drives.
   > Again, running sync directly is correct here. You replaced the parity drive intentionally, the data drives are untouched and healthy, and you know exactly what state the array is in.

5. Once sync completes, you are fully protected again.

---

### Quick Reference — Which Scenario Am I In?

| Failed drive | Data at risk? | SnapRAID fix needed? | Urgency |
|---|---|---|---|
| NVMe (cache) | Last ~24h of writes only | No | Low — replace at convenience |
| HDD data drive (Bay 1/3/4) | Files on that drive | Yes — run `fix` before `sync` | High — replace and recover ASAP |
| HDD parity drive (Bay 2) | None | No — just replace and resync | Medium — you're unprotected until done |

---

## Adding Drives Later (Future Expansion)

When you add drives to Bays 3 and 4, you only ever touch **Pool 1**. Pool 2 never changes — it still points to the NVMe and `/srv/mergerfs/data`, and Pool 1 now just has more drives behind that mount point. From the outside, nothing changes — the same share paths, the same pool mount, the same nightly routine.

**Steps:**

1. **Check the SMART status of the new drive** before trusting it with data — same as Phase 6.

2. **Format the new drive with Btrfs** (same as Phase 8):
   ```bash
   mkfs.btrfs /dev/sdX -L "data2"   # or data3 for the third drive
   ```

3. **Mount it in OMV** (`Storage` → `File Systems`)

4. **Add it to Pool 1 only** (`Storage` → `MergerFS` → edit the `data` pool → add the new drive)
   Do not touch Pool 2 (`pool`) — it automatically benefits because Pool 1 now has more drives behind it.

5. **Add it to SnapRAID as a data drive** (`Storage` → `SnapRAID` → add drive)

6. **Update the mount guard script** — add the new drive's UUID path to the `DRIVES` array in `/usr/local/bin/snapraid-safe-sync.sh`. If you skip this, the mount check won't protect the new drive from triggering a blind sync.

7. **Run a manual sync** to compute parity for the new drive:
   ```bash
   snapraid sync
   ```
   This is safe to run directly — you're adding a known-good drive intentionally, in a controlled situation.

8. **Update the hd-idle config** to include the new drive:
   Add `-a /dev/disk/by-label/data2 -i 1800 \` to `/etc/default/hd-idle` and restart the service:
   ```bash
   systemctl restart hd-idle
   ```

From this point, `epmfs` in Pool 1 will naturally route new writes and cache flushes to whichever HDD has the most free space — including the new one. No manual rebalancing needed.

**Parity capacity:** the existing Bay 2 parity drive (18 TB) can protect up to 3 data drives (Bays 1, 3, and 4), all 18 TB or smaller. If you add a fourth data drive larger than 18 TB, you would need to upgrade the parity drive to match. *Search: snapraid parity drive size requirement*

---

*Guide written for OMV 7 (Sandworm) on Debian 12 Bookworm. Plugin names and UI paths may shift slightly in future OMV versions.*
