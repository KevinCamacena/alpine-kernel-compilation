# Playbook: Compiling a Custom Linux Kernel on Alpine

**Target Environment:** Alpine Linux VM (musl libc, syslinux/extlinux)
**Kernel Version:** 7.0.0-rc6

This guide documents the end-to-end process for compiling a mainline Linux kernel on an Alpine Linux host. It includes specific mitigations for building within a `musl` libc environment and bypassing official Alpine builder cryptographic signatures.

---

## Prerequisites

Ensure the `community` repository is enabled in your package manager configuration.

```bash
# Enable community repository if not already present
doas vi /etc/apk/repositories

# Update package index
doas apk update
```

Install all required build tools, replacing Alpine's default BusyBox applets with GNU utilities required by the kernel build scripts, along with math libraries for GCC plugins.

```bash
doas apk add \
  build-base linux-headers bash \
  flex bison bc perl \
  openssl-dev elfutils-dev ncurses-dev zstd-dev xz tar wget \
  coreutils findutils diffutils gawk grep sed \
  pkgconf python3 argp-standalone pahole \
  gmp-dev mpfr-dev mpc1-dev \
  fastfetch
```

## Step 1: Preparation & Configuration

Create a build directory, extract the kernel sources, and clone the configuration from the current running kernel.

```bash
# Check for kernel versions here: https://github.com/torvalds/linux/tags
wget -qO- https://github.com/torvalds/linux/archive/refs/tags/v7.0-rc6.tar.gz
# Set up directories and extract sources
mkdir -p ~/.build-rc
tar -xf v7.0-rc6.tar.gz -C ~/.build-rc/
cd ~/.build-rc/linux-7.0-rc6/

# Copy the host's existing kernel configuration
cp /boot/config-$(uname -r) .config
```

## Step 2: Patching Configurations for Alpine

When cloning an official Alpine kernel configuration, hardcoded builder keys and musl libc incompatibilities must be patched before compiling.

1. Clear Hardcoded Cryptographic Keys
Clear the absolute paths to the official Alpine maintainer's signing keys to prevent the build from halting, and set the module signing key to auto-generate an ephemeral key.

```bash
./scripts/config --set-str SYSTEM_TRUSTED_KEYS ""
./scripts/config --set-str SYSTEM_REVOCATION_KEYS ""
./scripts/config --set-str MODULE_SIG_KEY "certs/signing_key.pem"

# Apply configuration changes
make olddefconfig
```

2. Fix Musl Libc __attribute_const__ Missing Definition
Patch the host tool insn_sanity.c to explicitly include the Linux compiler definitions, bypassing musl libc's strict compliance which lacks this GNU macro.

```bash
sed -i '1i #include <linux/compiler.h>' arch/x86/tools/insn_sanity.c
```

Step 3: Compilation
Compile the kernel using all available CPU cores.

```bash
make -j$(nproc)
```

Step 4: Installation
Install the compiled kernel modules to /lib/modules/ and manually copy the core kernel files to the boot directory.

```bash
# Install modules
doas make modules_install

# Copy the kernel image, system map, and configuration
doas cp arch/x86/boot/bzImage /boot/vmlinuz-7.0.0-rc6
doas cp System.map /boot/System.map-7.0.0-rc6
doas cp .config /boot/config-7.0.0-rc6
```

## Step 5: Bootloader Configuration (Syslinux/Extlinux)
Alpine requires generating an initial ramdisk (initramfs) for the new modules and manually adding the boot entry to extlinux.conf.

1. Generate the Initramfs

```bash
doas mkinitfs -o /boot/initramfs-7.0.0-rc6 7.0.0-rc6
```

2. Add Bootloader Entry

```bash
doas vi /boot/extlinux.conf
```

Action: Copy your existing boot entry block (e.g., LABEL lts) and paste it below. Update the label, menu label, kernel, and initrd variables to point to the new -7.0.0-rc6 files. Crucial: Preserve the existing APPEND line exactly as it is to ensure the root filesystem UUID remains correct.

Optionally, change the DEFAULT parameter at the top of the file to point to your new label to boot it automatically.

_Should look like this_:

```.conf
DEFAULT custom
PROMPT 0
MENU TITLE Alpine/Linux Boot Menu
MENU HIDDEN
MENU AUTOBOOT Alpine will be booted automatically in # seconds.
TIMEOUT 10
LABEL lts
  MENU DEFAULT
  MENU LABEL Linux lts
  LINUX vmlinuz-lts
  INITRD initramfs-lts
  APPEND root=UUID=xxx modules=sd-mod,usb-storage,ext4 quiet rootfstype=ext4

MENU SEPARATOR

LABEL custom
  MENU LABEL Linux 7.0.0-rc6
  LINUX vmlinuz-7.0.0-rc6
  INITRD initramfs-7.0.0-rc6
  APPEND root=UUID=xxx modules=sd-mod,usb-storage,ext4 quiet rootfstype=ext4

MENU SEPARATOR
```

Step 6: Verify
Reboot the system and verify the new kernel is running.

```bash
doas reboot

# After reboot, confirm the running version
fastfetch
```