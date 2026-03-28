# Linux Kernel Development with virtme-ng (vng)

This document summarizes the workflow for developing and testing the Linux kernel using `virtme-ng` (vng), focusing on filesystem and network development.

## 1. Initial Setup & Build

Start with a clean environment to avoid configuration conflicts.

```bash
# Clean previous build artifacts
make mrproper
```

Build the kernel. `vng` generates a minimal configuration optimized for virtualization.

```bash
# Basic build (might lack specific drivers)
vng --build
```

## 2. enabling Essential Drivers (Filesystems, etc.)

The default `vng` config is minimal. For filesystem development (e.g., ext4) and a usable read-write environment (overlayfs), you must enable specific config options.

**Rebuild with required modules:**

```bash
vng --build \
  --configitem CONFIG_EXT4_FS=y \
  --configitem CONFIG_OVERLAY_FS=y \
  --configitem CONFIG_TMPFS=y \
  --configitem CONFIG_DEVTMPFS=y \
  --configitem CONFIG_SHMEM=y
```

*   `CONFIG_EXT4_FS`: Enables support for ext4 formatted disks.
*   `CONFIG_OVERLAY_FS`: Required by `vng` to provide writable directories (like `/home`) via overlayfs. Without this, the root filesystem is purely read-only.

## 3. Creating and Using a Test Disk

To test filesystem changes, create a raw disk image on your host.

```bash
# Create a directory for images
mkdir -p ~/kernel-dev

# Create a 1GB raw disk image filled with zeros
dd if=/dev/zero of=~/kernel-dev/testdisk.img bs=1M count=1024 status=progress
```

## 4. Running the Kernel

Launch the VM with the disk image attached.

```bash
# Run with the disk image
vng --disk ~/kernel-dev/testdisk.img
```

*   **Networking:** Add `--network user` for user-mode networking (outbound internet access).
    ```bash
    vng --disk ~/kernel-dev/testdisk.img --network user
    ```

## 5. Workflow Inside the VM

Once logged into the VM:

1.  **Identify the Disk:** It usually appears as `/dev/vda` (since the root fs is a 9p/virtio-fs share, not a block device).
    ```bash
    lsblk
    ```
2.  **Format (First run only):**
    ```bash
    mkfs.ext4 /dev/vda
    ```
3.  **Mount:**
    Create a mount point in a writable directory (e.g., inside `~` or `/tmp`).
    ```bash
    mkdir -p ~/test_mount
    sudo mount /dev/vda ~/test_mount
    ```
4.  **Test:**
    Perform your file operations in `~/test_mount`. Data persists to `~/kernel-dev/testdisk.img` on the host.

## 6. Automated Testing with Scripts

You can execute scripts immediately upon startup using the `--exec` flag.

**Example Python Script (`test_script.py`):**
```python
import os
import subprocess

print("Starting automated test...")
try:
    os.makedirs("/root/mnt", exist_ok=True)
    subprocess.check_call(["mount", "/dev/vda", "/root/mnt"])
    print("Disk mounted successfully!")
    # Perform tests...
except Exception as e:
    print(f"Test failed: {e}")
```

**Run the test:**
```bash
vng --disk ~/kernel-dev/testdisk.img --exec "python3 ./test_script.py"
```

## Troubleshooting

*   **Read-only file system errors:** Ensure `CONFIG_OVERLAY_FS=y` is set in the kernel config.
*   **Unknown filesystem type:** Ensure the relevant filesystem driver (e.g., `CONFIG_EXT4_FS`) is enabled.
*   **Permission denied (build):** Run `make mrproper` and rebuild if you encounter strange permission errors during compilation.
