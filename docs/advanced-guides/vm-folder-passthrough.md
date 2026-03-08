# VirtioFS host folder into VM with fstab automount

::: info GOAL
Share a host folder from UGOS Pro into a Linux VM using virtiofs, and have it mounted via `/etc/fstab` without breaking boot.
We also use `bindfs` inside the VM so your VM user has correct read/write permissions, and files created inside the VM map correctly to UGOS's user and "admin" group (GID 10).
:::

> [!WARNING]
> This guide was written for UGOS Pro only, and was last tested with the version 1.13.x

## 1. On the UGOS host: add virtiofs to the VM

1. SSH into the UGOS host and become root:

    ```bash
    ssh root@<ugos-host>
    ```

    - or ssh \<username>@\<ugos-host> then:

    ```bash
    sudo -i   # if sudo exists, otherwise use whatever UGOS provides for root
    ```

2. Find your VM name in libvirt:

    ```bash
    virsh list --all
    ```

    > [!NOTE]
    > The Name will be in the format of a UUID, e.g. "f229f5ce-b904-4027-a50b-ba24b1d8e2ea"

3. Dump the VM XML to a temp file:

    ```bash
    virsh dumpxml <vm-name> > /tmp/vm-config.xml
    ```

4. Edit `/tmp/vm-config.xml` and add **shared memory backing** at top level
   (as a direct child of `<domain>`, not inside `<devices>`):

    ```xml
    <memoryBacking>
        <access mode='shared'/>
        <source type='memfd'/>
    </memoryBacking>
    ```

    Place it anywhere before `<devices>`, as long as it’s a sibling of `<memory>`, `<vcpu>`, `<devices>`, etc.

5. In the same XML file, inside the `<devices>` section, add a `filesystem` block:
   
    > [!NOTE]
    > Modify the `/volume1/projects` path below to the full path of your shared folder. You can rename the 'tag' in the `<target dir=...>` field too; remember this tag for later.

    ```xml
    <devices>
        ...
        <filesystem type='mount' accessmode='passthrough'>
        <driver type='virtiofs'/>
        <source dir='/volume1/projects'/>
        <target dir='projects-fs'/>
        </filesystem>
        ...
    </devices>
    ```

    - Replace `/volume1/projects` with the host path you want to mount.
    - The `<target dir='projects-fs'/>` string is the **tag** the VM will mount. Remember this tag exactly (case‑sensitive).

6. Redefine the VM from the modified XML:

    ```bash
    virsh define /tmp/vm-config.xml
    ```

7. Restart the VM so the new config is applied:

    ```bash
    virsh shutdown <vm-name>
    virsh start <vm-name>
    ```

## 2. Inside the VM: Install bindfs and create mount points

Because UGOS forces `passthrough` access mode for virtiofs, the mount will show up owned by `root:root` with strict permissions. We use `bindfs` to remap this safely for your normal user without modifying the host.

1. Log into the VM (console or SSH).

2. Install `bindfs` (Debian/Ubuntu example):

    ```bash
    sudo apt update
    sudo apt install bindfs
    ```

3. Create two mount points (one raw/hidden, and one for your user):

    ```bash
    sudo mkdir -p /mnt/.projects-raw
    sudo mkdir -p /mnt/projects
    ```

    - `projects-fs` must exactly match the `<target dir='...'/>` tag in the VM XML.
    - If this works and `/mnt/projects` shows the host files, proceed to edit `/etc/fstab` for automount, otherwise fix the issue before continuing.

## 3. Configure fstab with virtiofs and bindfs

We will mount the raw virtiofs share to the hidden folder, then use `bindfs` to present it to the final folder with the correct user permissions.


1. Edit `/etc/fstab` in the **VM**:

    ```bash
    sudo nano /etc/fstab
    ```

2. Add these two lines. (Assume your VM user's UID is `1000`, and UGOS expects files to be created with group "admin", which is GID `10`):

    > [!NOTE]
    > It works best if the USERNAME inside your VM matches the USERNAME in UGOS that you want to have ownership of the files created inside this folder.
    
    > [!NOTE]
    > If you don't know the UID of your VM user, run this command: `id $USER`. The admin GID of 10 is required by UGOS, please don't change that.

    ```fstab
    # 1. Mount raw virtiofs to a hidden staging folder
    projects-fs  /mnt/.projects-raw  virtiofs  nofail,x-systemd.automount,x-systemd.device-timeout=10  0  0

    # 2. Use bindfs to remap ownership and permissions to your user
    bindfs#/mnt/.projects-raw  /mnt/projects  fuse  force-user=1000,force-group=1000,create-as-user,create-for-group=10,chown-ignore,chgrp-ignore,perms=0770,nofail,x-systemd.requires=/mnt/.projects-raw  0  0
    ```

    **What these flags do:**
    - `x-systemd.automount`: Defers mounting until first access so it doesn't break VM boot timing.
    - `force-user=1000,force-group=1000,perms=0770`: Makes `/mnt/projects` completely readable/writable by your VM user (UID/GID 1000).
    - `create-as-user,create-for-group=10`: Ensures files created by your VM user map correctly to your host user and the UGOS `admin` group (GID 10).
    - `chown-ignore,chgrp-ignore`: Prevents permission errors when the VM OS attempts to change ownership on the FUSE mount.

3. Reload systemd units so the new fstab is recognized:

    ```bash
    sudo systemctl daemon-reload
    ```

## 4. Verify automount works

1. Trigger the automount by accessing the raw folder, then mount the bindfs layer:

    - *(Alternatively, just reboot the VM).*

    ```bash
    ls /mnt/.projects-raw
    sudo mount /mnt/projects
    ```

2. Verify ownership:
    ```bash
    ls -ld /mnt/projects
    ```

    - It should now show `drwxrwx--- youruser yourgroup ... /mnt/projects`.

3. Optional: confirm with `mount`:

    ```bash
    mount | grep projects-fs
    ```

    - or:

    ```bash
    mount | grep /mnt/projects
    ```

    - You should see a line showing `projects-fs` mounted on `/mnt/projects` with type `virtiofs`.

4. Test file creation:

    As your non-root user, create a file:

    ```bash
    touch /mnt/projects/test-file.txt
    ```

    - If you check this file on the UGOS host via SSH (`ls -l /volume1/projects/test-file.txt`), it will correctly show as owned by your UGOS user and the `admin` group.

## 5. Common gotchas to double‑check

- The tag in fstab (`projects-fs`) must exactly match the `<target dir='...'/>` tag in the VM XML.
- If you ever edit the VM via the UGOS GUI, you may lose the `<filesystem>` block. If the mount suddenly fails, re‑add the XML block and restart the VM.

::: info Credit
This guide was created by [Drauku](https://github.com/Drauku)
:::