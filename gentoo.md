# Gentoo Guest Experiments


### Gentoo Guest VM Template
  * Create a vm: `sudo vm create -t gentoo -s 32g gentoo-test`
  * Check it's there: `sudo vm iso list`
  * Fetch the gentoo livecd iso: `sudo vm iso http://distfiles.gentoo.org/releases/amd64/autobuilds/20180325T214502Z/install-amd64-minimal-20180325T214502Z.iso`
  * Open tmux: `screen`
  * In pane 0: `sudo vm install gentoo-test install-amd64-minimal-20180325T214502Z.iso` and `Ctrl + a c` to pane 1
  * In pane 1: `sudo vm console gentoo-test` and `Ctrl + a n` to pane 0

### Gentoo Base Install (stage1)
#### GPT variant, doesn't boot
Run parted to partition disks `parted -a optimal /dev/sda`
  * `mklabel gpt`
  * `unit mib`

Create grub partition
  * `mkpart primary 1 3`
  * `name 1 grub`
  * `set 1 bios_grub on`

Create boot partition
  * `mkpart primary 3 131`
  * `name 2 boot`

Create swap (2 GB) partition
  * `mkpart primary 131 2179`
  * `name 3 swap`

Create rootfs partition
  * `mkpart primary 2179 -1`
  * `name 4 rootfs`
  * check with `print`
  * and `quit`

Format the partitions, enable swap and mount:
  * `mkfs.ext2 /dev/sda2`
  * `mkfs.ext4 /dev/sda4`
  * `mkswap /dev/sda3`
  * `swapon /dev/sda3`
  * `mount /dev/sda4 /mnt/gentoo`

#### MSDOS variant, should boot:

    (parted) mklabel msdos
    (parted) unit mib
    (parted) mkpart primary 1 128
    (parted) set 1 boot on
    (parted) mkpart primary 128 2146
    (parted) mkpart primary 2146 -1
    (parted) p
      Model: ATA BHYVE SATA DISK (scsi)
      Disk /dev/sda: 34.4GB
      Sector size (logical/physical): 512B/131072B
      Partition Table: msdos
      Disk Flags:

      Number  Start   End     Size    Type     File system  Flags
       1      1049kB  134MB   133MB   primary               boot
       2      134MB   2250MB  2116MB  primary
       3      2250MB  34.4GB  32.1GB  primary
    (parted) q

  Format the partitions, enable swap and mount:

    mkfs.ext2 /dev/sda1
    mkfs.ext4 /dev/sda3
    mkswap /dev/sda2
    swapon /dev/sda2
    mount /dev/sda3 /mnt/gentoo


### Gentoo Base Install (stage3)
  * `cd /mnt/gentoo`
  * Download stage3, I am going with hardened-nomultilib `wget http://distfiles.gentoo.org/releases/amd64/autobuilds/current-stage3-amd64-hardened/stage3-amd64-hardened+nomultilib-20180325T214502Z.tar.xz`
  * Download hash: `wget http://distfiles.gentoo.org/releases/amd64/autobuilds/current-stage3-amd64-hardened/stage3-amd64-hardened+nomultilib-20180325T214502Z.tar.xz.DIGESTS`
  * Download asc:
  `wget http://distfiles.gentoo.org/releases/amd64/autobuilds/current-stage3-amd64-hardened/stage3-amd64-hardened+nomultilib-20180325T214502Z.tar.xz.DIGESTS.asc`
  * Verify hash: `sha512sum stage3-amd64-hardened+nomultilib-20180325T214502Z.tar.xz` and compare it with .DIGEST file
  * Download gpg keys: `gpg --keyserver pgp.mit.edu --receive-keys 0x825533CBF6CD6C97 0xDB6B8C1F96D8BF6D 0x9E6438C817072058 0xBB572E0E2D182910 0x0838C26E239C75C4 0x6DC226AAD8BA32AA 0xBB1D301B7DDAD20D` with keys from https://wiki.gentoo.org/wiki/Project:RelEng
  * Verify signature: `gpg --verify stage3-amd64-hardened+nomultilib-20180325T214502Z.tar.xz.DIGESTS.asc`
  * Extract tarball: `tar xpf stage3-amd64-hardened+nomultilib-20180325T214502Z.tar.xz --xattrs-include='*.*' --numeric-owner`
  * Compile options: `nano -w /mnt/gentoo/etc/portage/make.conf` and put

        CFLAGS="-march=native -O2 -pipe"
        CXXFLAGS="${CFLAGS}"
        MAKEOPTS="-j5"

  * Choose portage mirror: `mirrorselect -i -o >> /mnt/gentoo/etc/portage/make.conf`
  * Create repos config: `mkdir --parents /mnt/gentoo/etc/portage/repos.conf`
  * Copy config from portage: `cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf`
  * Copy resolv.conf: `cp --dereference /etc/resolv.conf /mnt/gentoo/etc/`

  Prepare for chroot:
  * `mount --types proc /proc /mnt/gentoo/proc`
  * `mount --rbind /sys /mnt/gentoo/sys`
  * `mount --make-rslave /mnt/gentoo/sys`
  * `mount --rbind /dev /mnt/gentoo/dev`
  * `mount --make-rslave /mnt/gentoo/dev`

  Chroot into the new system:
  * `chroot /mnt/gentoo /bin/bash`
  * `source /etc/profile`
  * `export PS1="(chroot) ${PS1}"`
  * Mount boot: (gpt:) `mount /dev/sda2 /boot` or (msdos) `mount /dev/sda1 /boot`
  * Update emerge: `emerge --sync`
  * Show profiles `eselect profile list`
  * Select profile: `eselect profile set hardened/linux/amd64/no-multilib`
  * Update @world: `emerge --ask --update --deep --newuse @world`
  * Clean: `emerge --depclean`
  * Set timezone: `echo "Europe/Berlin" > /etc/timezone`
  * Set timezone-data: `emerge --config sys-libs/timezone-data`
  * Configure locales: `nano -w /etc/locale.gen` uncomment the one you need
  * locale-gen: `locale-gen`
  * system wide locale: `eselect locale list` and `eselect locale set 3`
  * check the locale: `env-update && source /etc/profile && export PS1="(chroot) $PS1"`

  Kernel build:
  * Get sources: `emerge --ask sys-kernel/gentoo-sources`
  * Genkernel install: `emerge --ask sys-kernel/genkernel`
  * `cd /usr/src/linux` and `make menuconfig`
  * Tune your kernel configuration, make sure to include the following:

        Processor type and features  --->
        [*] Linux guest support --->
          [*] Enable Paravirtualization code
          [*] KVM Guest support (including kvmclock)
        Device Drivers  --->
        Virtio drivers  --->
          <*>   PCI driver for virtio devices
        [*] Block devices  --->
          <*>   Virtio block driver
        [*] Network device support  --->
          <*>   Virtio network driver
        SCSI device support  --->
          [*] SCSI low-level drivers  --->
              [*]   virtio-scsi support


  * Check the `.config` file and enable everything with "VIRTIO" in it.
  * `cp .config vm.config`
  * `genkernel --kernel-config=vm.config --install all`

  * Edit: `nano -w /etc/fstab` with the partitions created

  Fstab (gpt version)
        /dev/sda2   /boot        ext2    defaults,noatime     0 2
        /dev/sda3   none         swap    sw                   0 0
        /dev/sda4   /            ext4    noatime              0 1

  Fstab (gpt version)
        /dev/sda1   /boot        ext2    defaults,noatime     0 2
        /dev/sda2   none         swap    sw                   0 0
        /dev/sda3   /            ext4    noatime              0 1


  Networking:
  * Set hostname `nano -w /etc/conf.d/hostname`
  * Set domainname `nano -w /etc/conf.d/net`

        # Set the dns_domain_lo variable to the selected domain name
        dns_domain_lo="g.hw.byte23.net"
  * Install netifrc `emerge --ask --noreplace net-misc/netifrc`
  * Up networks on boot: `cd /etc/init.d && ln -s net.lo net.eth0 && rc-update add net.eth0 default`

  Final preparations:
  * Set root password: `passwd`
  * Set keymap (to "de" in my case) `nano -w /etc/conf.d/keymaps`
  * Install sysklogd: `emerge --ask app-admin/sysklogd`
  * rc for syslog: `rc-update add sysklogd default`
  * cronie: `emerge --ask sys-process/cronie`
  * rc for cronie: `rc-update add cronie default`
  * mlocate: `emerge --ask sys-apps/mlocate`
  * sshd: `rc-update add sshd default`
  * dhcp-client: `emerge --ask net-misc/dhcpcd`

  Bootloader:
  * grub2: `emerge --ask --verbose sys-boot/grub:2`
  * install: `grub-install /dev/sda`
  * configure: `grub-mkconfig -o /boot/grub/grub.cfg`

  User:
  * create user: `useradd -m -G users,wheel -s /bin/bash tobi`
  * set password: `passwd tobi`
  * copy ssh-key: `echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDXXbg63Qn/SHh8XIhn2QYzhuoJkK48d26Z8pNnp4a1zs2RHjQVtwnG+eUggh3ywnUSDxCTiN6CKjfZ2GOuwKLyLyGE5Dd8XDowCn5WOenFxrZn2EN4GTDX7oxQei154o3ISuMUNVv2t067IXVHwORa9sCTjGwRwoc/mF5PdlklMWxKV2Y3ZmqMoX3k+/QaIwO5fuOaIxIyVu4FDekNIg+0cqgH99B/Zm/lS/SZgfoNPmcJzCFiQEo4WC95gn+NdTJdFAwYjYsQfiSkJlEtFtp2XeP7tCgS3289T7Kzd8eF+kzRP4U0eFJ9YKwupF+V7uXhnOxlpzJJTUhTdn+qUKFZ tobi@gemini.b.hw.byte23.net" >> ~tobi/.ssh/authorized_keys`

  Finish:
  * `exit`
  * `cd`
  * `umount -l /mnt/gentoo/dev{/shm,/pts,}`
  * `umount -R /mnt/gentoo`
  * `poweroff`


  # Rewrite this to vm-behyve:

  ### Alpine Guest VM Template
  * Create a vm: `sudo chyves alpine create 32g`
  * Set parameters: `sudo chyves alpine set cpu=4 ram=8g loader=grub-bhyve os=alpine`
  * Check it's there: `sudo chyves iso list`
  * Download ISO: `fetch http://dl-cdn.alpinelinux.org/alpine/v3.7/releases/x86_64/alpine-virt-3.7.0-x86_64.iso`
  * Import ISO: `sudo chyves iso import alpine-virt-3.7.0-x86_64.iso` delete it afterwards.
  * Open tmux: `tmux`
  * In pane 0: `sudo chyves alpine console` and `Ctrl + b c` to pane 1
  * In pane 1: `sudo chyves alpine start alpine-virt-3.7.0-x86_64.iso` and `Ctrl + b n` to pane 0


  ### Gentoo Guest VM Template
    * Create a vm: `sudo chyves gentoo create 32g`
    * Set parameters: `sudo chyves gentoo set cpu=4 ram=8g loader=grub-bhyve os=gentoo`
    * Check it's there: `sudo chyves iso list`
    * Fetch the gentoo livecd iso: `fetch http://distfiles.gentoo.org/releases/amd64/autobuilds/20180325T214502Z/install-amd64-minimal-20180325T214502Z.iso`
    * Rename it: `mv install-amd64-minimal-20180325T214502Z.iso gentoo-minimal-stage1.iso`
    * Import the iso: `sudo chyves iso import gentoo-minimal-stage1.iso && rm gentoo-minimal-stage1.iso`
    * Open tmux: `tmux`
    * In pane 0: `sudo chyves gentoo console` and `Ctrl + b c` to pane 1
    * In pane 1: `sudo chyves gentoo start gentoo-minimal-stage1.iso` and `Ctrl + b n` to pane 0
