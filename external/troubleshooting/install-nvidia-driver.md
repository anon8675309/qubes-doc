---
layout: doc
title: How to Install an Nvidia Driver
permalink: /doc/install-nvidia-driver/
redirect_from:
- /en/doc/install-nvidia-driver/
- /doc/InstallNvidiaDriver/
- /wiki/InstallNvidiaDriver/
---

# Nvidia proprietary driver installation

You can use rpm packages from rpmfusion, or you can build the driver yourself.

## Word of Caution 

Proprietary (NVIDIA/AMD) drivers are known to be sometimes highly problematic, or completely unsupported. 
Radeon driver support is prebaked in the Qubes kernel (v4.4.14-11) but only versions 4000-9000 give or take.
Support for newer cards is limited until AMDGPU support in the 4.5+ kernel, which isn't released yet for Qubes. 

Built in Intel graphics, Radeon graphics (between that 4000-9000 range), and perhaps some prebaked NVIDIA card support that I don't know about. Those are your best bet for great Qubes support.

If you do happen to get proprietary drivers working on your Qubes system (via installing them), please take the time to go to the 
[Hardware Compatibility List (HCL)](/doc/hcl/#generating-and-submitting-new-reports )
Add your computer, graphics card, and installation steps you did to get everything working.

# Instructions
The general process will be to get the kernel headers for the kernel being used in dom0, build the kernel module (.ko file), copy said kernel module to dom0 and load it.  Instructions were largely taken from [Issue 2908](https://github.com/QubesOS/qubes-issues/issues/2908).

## Kernel Headers
After several hours, the author of this document was unable to obtain the kernel headers for 4.19.94-1.pvops.qubes.x86_64 on a Qube (tried Fedora 25/27/28/29/30).  As a result, dom0 is used to download the .rpm containing your kernel headers.

For Qubes 3.2:
`sudo qubes-dom0-update --action=download kernel-devel`

For Qubes 4.0:
`sudo qubes-dom0-update --downloadonly kernel-devel`
If the kernel-devel package was already downloaded in dom0, the above command will error out and you should use the command below instead:
`sudo qubes-dom0-update --downloadonly --action=reinstall kernel-devel`

## Building the Kernel Module (Debian to the Rescue)
Next, we are going to use Debian to build the kernel module.  Maybe there's a way to do it with Fedora, and if anyone is ever able to figure that out, please document it here.  In the meantime, we'll use Debian, because it's what works.

Create a Debian-based Qube.  In this example, we are using Qubes 4.0.3rc1 and our Qube is named "deb" and is based on the debian-10 template.  In dom0, we copy the kernel headers RPM file to our Debian VM.

`qvm-copy-to-vm dev /var/lib/qubes/updates/rpm/kernel-devel-$(uname -r).rpm`

In our Debian VM, we install alien to convert the .rpm to a .deb and install it.

```
sudo aptitude install alien
sudo alien kernel-devel-4.19.84-1.pvops.qubes.x86_64.rpm
sudo dpkg -i ./kernel-devel_4.19.84-2_amd64.deb
```

Next, we download the closed source nVidia driver [from nVidia's site](https://www.nvidia.com/Download/index.aspx).  In this example, NVIDIA-Linux-x86_64-440.44.run is used.  The next steps will be to extract the files from nVidia's package (do not install any extra things such as 32-bit compatibility libraries, libglvnd files or modify X configs), and build the kernel module.

```
sudo ./NVIDIA-Linux-x86_64-440.44.run --ui=none --no-x-check --keep --no-nouveau-check --no-kernel-module
cd NVIDIA-Linux-x86_64-440.44/kernel/
sudo IGNORE_CC_MISMATCH=1 make
cp nvidia.ko /home/user
```

Note: The IGNORE_CC_MISMATCH is required if using a different version of the compiler than what was used to compile the kernel.  Getting a specific version of Debian may allow getting an old version of gcc (which matches that which was used to create the kernel) and the IGNORE_CC_MISMATCH=1 part can be omitted.  This would be preferred, and if a specific solution is found, it should be documented here.

Side note: nVidia's kernel module wouldn't compile out of the box (error about "latent_entropy" not being define), its Makefile doesn't respect CFLAGS, and the author of this document was unable to successfully hack the Makefile to work around this, so the kernel headers were modified to achieve the same effect.  The patch is below, and this may not have been necessary if an older version of GCC was used (the same version which was used to compile the kernel, 6.4.1 in this case, instead of 8.3.0-6 or 7.4.0-6).  This **should** be a safe modification, since it should just be linking to the function and care not for how the kernel actually implements it.
```
--- /usr/lib/modules/4.19.94-1.pvops.qubes.x86_64/build/include/linux/random.h.bak	2020-01-19 15:26:30.666000000 -0600
+++ /usr/lib/modules/4.19.94-1.pvops.qubes.x86_64/build/include/linux/random.h	2020-01-19 15:26:53.370000000 -0600
@@ -20,7 +20,7 @@
 
 extern void add_device_randomness(const void *, unsigned int);
 
-#if defined(CONFIG_GCC_PLUGIN_LATENT_ENTROPY) && !defined(__CHECKER__)
+#if defined(CONFIG_GCC_PLUGIN_LATENT_ENTROPY) && !defined(__CHECKER__) && 0
 static inline void add_latent_entropy(void)
 {
 	add_device_randomness((const void *)&latent_entropy,
```

## Loading Kernel Module in dom0
Now we need to get the kernel module into dom0 where it can be loaded.

```
qvm-run --pass-io deb 'cat /home/user/nvidia.ko' > nvidia.ko
chmod +x nvidia.ko
sudo insmod ./nvidia.ko
```

Sadly, even with these updated instructions, the kernel module isn't getting loaded.  The author has spent about 5 hour on this and is giving up and buying an AMD video card (as the warnings suggested).  Hopefully this is at least getting the documentation to be closer to correct and someone with more patients will come along, figure it out and update the documentation again with a solution.

Below is the current error that is preventing the kernel module from being loaded:
```
$ sudo insmod /lib/modules/4.19.94-1.pvops.qubes.x86_64/nvidia.ko
insmod: ERROR: could not insert module /lib/modules/4.19.94-1.pvops.qubes.x86_64/nvidia.ko: Unknown symbol in module
$ sudo modinfo /lib/modules/4.19.94-1.pvops.qubes.x86_64/nvidia.ko
filename:       /lib/modules/4.19.94-1.pvops.qubes.x86_64/nvidia.ko
alias:          char-major-195-*
version:        440.44
supported:      external
license:        NVIDIA
srcversion:     76C1A18886D409B7BFE6518
alias:          pci:v000010DEd*sv*sd*bc03sc02i00*
alias:          pci:v000010DEd*sv*sd*bc03sc00i00*
depends:        ipmi_msghandler
retpoline:      Y
name:           nvidia
vermagic:       4.19.94-1.pvops.qubes.x86_64 SMP mod_unload 
parm:           NvSwitchRegDwords:NvSwitch regkey (charp)
parm:           NVreg_Mobile:int
parm:           NVreg_ResmanDebugLevel:int
parm:           NVreg_RmLogonRC:int
parm:           NVreg_ModifyDeviceFiles:int
parm:           NVreg_DeviceFileUID:int
parm:           NVreg_DeviceFileGID:int
parm:           NVreg_DeviceFileMode:int
parm:           NVreg_InitializeSystemMemoryAllocations:int
parm:           NVreg_UsePageAttributeTable:int
parm:           NVreg_MapRegistersEarly:int
parm:           NVreg_RegisterForACPIEvents:int
parm:           NVreg_EnablePCIeGen3:int
parm:           NVreg_EnableMSI:int
parm:           NVreg_TCEBypassMode:int
parm:           NVreg_EnableStreamMemOPs:int
parm:           NVreg_EnableBacklightHandler:int
parm:           NVreg_RestrictProfilingToAdminUsers:int
parm:           NVreg_PreserveVideoMemoryAllocations:int
parm:           NVreg_DynamicPowerManagement:int
parm:           NVreg_EnableUserNUMAManagement:int
parm:           NVreg_MemoryPoolSize:int
parm:           NVreg_KMallocHeapMaxSize:int
parm:           NVreg_VMallocHeapMaxSize:int
parm:           NVreg_IgnoreMMIOCheck:int
parm:           NVreg_NvLinkDisable:int
parm:           NVreg_RegisterPCIDriver:int
parm:           NVreg_RegistryDwords:charp
parm:           NVreg_RegistryDwordsPerDevice:charp
parm:           NVreg_RmMsg:charp
parm:           NVreg_GpuBlacklist:charp
parm:           NVreg_TemporaryFilePath:charp
parm:           NVreg_AssignGpus:charp
```

At this point the kernel module should be loaded in the kernel.  However, we still need to set it up to automatically load on the next boot and block out the `nouveau` driver.

## Updating Bootloader
The next step is to blacklist the nouveau module so it doesn't try to take priority over the nVidia driver we want to use.  To do this, we add a parameter to the kernel when starting dom0.

Edit /etc/default/grub:

~~~
GRUB_CMDLINE_LINUX="quiet rhgb nouveau.modeset=0 rd.driver.blacklist=nouveau video=vesa:off"
~~~

Regenerate grub configuration:

~~~
grub2-mkconfig -o /boot/grub2/grub.cfg
~~~

Reboot.

# Legacy Instructions
This section contains older documentation which no longer works with modern releases of Fedora (Fedora 30 at the time of writing).

## RpmFusion packages

There are rpm packages with all necessary software on rpmfusion. The only package you have to compile is the kernel module (but there is a ready built src.rpm package).

### Download packages

You will need any Fedora system 18 to download and build packages. You can use Qubes AppVM for it, but it isn't necessary. To download packages from rpmfusion - add this repository to your yum configuration (instructions are on their website). Then download packages using yumdownloader:

~~~
yumdownloader --resolve xorg-x11-drv-nvidia
yumdownloader --source nvidia-kmod
~~~

### Build kernel package

You will need at least kernel-devel (matching your Qubes dom0 kernel), rpmbuild tool and kmodtool, and then you can use it to build the package.  The example below uses the 260 driver, but different hardware will require different versions of the driver.  There's [a guide](https://www.if-not-true-then-false.com/2015/fedora-nvidia-guide/) to help you determine the correct one.

~~~
sudo dnf install kernel-devel rpm-build kmodtool
rpmbuild --nodeps -D "kernels `uname -r`" --rebuild nvidia-kmod-260.19.36-1.fc13.3.src.rpm
~~~

In the above command, replace `uname -r` with kernel version from your Qubes dom0 (if something goes wrong, see the [Manual Installation](#manual-installation) instructions below).  If everything went right, you have now complete packages with nvidia drivers for the Qubes system.  Transfer them to dom0 (see [Copying from (and to) dom0](https://www.qubes-os.org/doc/copy-from-dom0/)) and install (using standard "yum install /path/to/file").

Alternative way to get the kernel drivers, is to run the commands below in a Qube running Fedora 25 (for Qubes 4.0.3; 4.19.x kernel).  The specific version is required because Fedora 30 doesn't have support for the 4.19 kernel, they've moved on to 5.3 and later.  If you don't have a Fedora 29 template, you can get one by running `sudo qubes-dom0-update qubes-template-fedora-29` in dom0 (see [TemplateVMs](https://www.qubes-os.org/doc/templates/) for more details).

~~~
# From your Fedora Qube that you'll be using to get the kernel...
sudo dnf update
# If `uname -r` reports the same kernel is running in your qube and dom0, you can use
# the command below, if not, replace $(uname -r) with the output from uname -r on dom0
sudo dnf install kernel-devel-$(uname -r) kernel-headers-$(uname -r) gcc make
# Oh no!  Fedora 30 doesn't seem to have anything for 4.19*
# FIXME (or use the instructions above)
~~~

Then you need to disable nouveau (normally it is done by install scripts from nvidia package, but unfortunately it isn't compatible with Qubes...).  See [Updating Bootloader](#updating-bootloader) section above.

## Manual installation

This process is quite complicated: First - download the source [from  site](https://www.nvidia.com/Download/index.aspx). Here "NVIDIA-Linux-x86\_64-260.19.44.run" is used. Copy it to dom0. Every next step is done in dom0.

See [this page](/doc/copy-to-dom0/) for instructions on how to transfer files to Dom0 (where there is normally no networking).

**WARNING**: Nvidia doesn't sign their files. To make it worse, you are forced to download them over a plaintext connection. This means there are virtually dozens of possibilities for somebody to modify this file and provide you with a malicious/backdoored file. You should realize that installing untrusted files into your Dom0 is a bad idea. Perhaps it might be a better idea to just get a new laptop with integrated Intel GPU? You have been warned.



### Userspace components

Install libraries, Xorg driver, configuration utilities. This can by done by nvidia-installer:

~~~
./NVIDIA-Linux-x86_64-260.19.44.run --ui=none --no-x-check --keep --no-nouveau-check --no-kernel-module
~~~

When prompted, do install NVIDIA's 32-bit compatibility libraries, overwrite existing libglvnd libraries, and automatically update your X configuration.

### Kernel module

You will need:

-   nvidia kernel module sources (left from previous step)
-   kernel-devel package installed
-   gcc, make, etc

This installation must be done manually, because nvidia-installer refused to install it on Xen kernel. Firstly ensure that kernel-devel package installed all needed files. This should consist of:

-   */usr/src/kernels/2.6.34.1-12.xenlinux.qubes.x86\_64*
-   */lib/modules/2.6.34.1-12.xenlinux.qubes.x86\_64/build* symlinked to the above directory
-   */usr/src/kernels/2.6.34.1-12.xenlinux.qubes.x86\_64/arch/x64/include/mach-xen* should be present (if not - take it from kernel sources)

If all the files are not there correct the errors manually. To build the kernel module, enter *NVIDIA-Linux-x86\_64-260.19.44/kernel* directory and execute:

~~~
make
IGNORE_XEN_PRESENCE=1 CC="gcc -DNV_VMAP_4_PRESENT -DNV_SIGNAL_STRUCT_RLIM" make -f Makefile.kbuild
mv /lib/modules/2.6.34.1-12.xenlinux.qubes.x86_64/kernel/drivers/video/nvidia.ko /lib/modules/2.6.34.1-12.xenlinux.qubes.x86_64/extra/
~~~

Ignore any errors while inserting nvidia.ko (at the end of make phase). 

### Disable nouveau:

~~~
cat /etc/modprobe.d/nouveau-disable.conf
# blacklist isn't enough...
install nouveau /bin/true
~~~

Add *rdblacklist=nouveau* option to /boot/grub/menu.lst (at the end of line containing *vmlinuz*).

### Configure Xorg

Finally, you should configure Xorg to use nvidia driver. You can use *nvidia-xconfig* or do it manually:

~~~
X -configure
mv /root/xorg.conf.new /etc/X11/xorg.conf
# replace Driver in Device section by "nvidia"
~~~

Reboot to verify all this works.

# Troubleshooting lack of video output during installation

Specifically, the notes below are aimed to help when the GRUB menu shows up fine, the installation environment starts loading, and then the display(s) go into standby mode. This is, typically, related to some sort of an issue with the kernel's KMS/video card modules.

## Initial setup.
*Note*: The steps below do *not* produce a fully-functional Qubes OS install. Rather, only a dom0 instance is functional, and there is no networking there. However, they can be used to gather data in order to troubleshoot video card issues and/or possible other basic kernel module issues.

1. Append `nomodeset ip=dhcp inst.nokill inst.vnc` to the kernel command line. Remove `rhgb` and `quiet` to see the kernel messages scroll by, which may help in further diagnostics.
   * If DHCP is not available on the installation network, the syntax becomes a bit more involved. The full list of variants is documented in the [Dracut Command-line parameters] (http://man7.org/linux/man-pages/man7/dracut.cmdline.7.html)
2. The VGA console should switch into the installer's multi-virtual-terminal display. VNC may take a number of minutes to start, please be patient.
   * Using the anaconda installer interface, switch to the "shell" TTY (ALT-F2), and use `ip a` command to display the IP addresses.
3. Using the Connect to the IP (remember the :1) using a VNC viewer.
4. Follow the installation UI. 
   * Since this won't be a usable install, skipping LUKS encryption is an option which will simplify this troubleshooting process.
   * Do *not* reboot at the end of the installation.
5. Once the installation completes, use the local VGA console switch to TTY2 via ALT-F2
   * Switch to the chroot of the newly-installed system via `chroot /mnt/sysinstall`
   * Set the root password (this will also enable the root account login)
   * Double-check that `/boot/grub2/grub.cfg` contains a `nomodeset` kernel parameter.
   * Exit out of the chroot environment (`exit` or CTRL-D)
6. Reboot

*Note* If the kernel parameters do *not* include `quiet` and `rhgb`, the kernel messages can easily obscure the LUKS passphrase prompt. Additionally, each character entered will cause the LUKS passphrase prompt to repeat onto next line. Both of these are cosmetic. The trade-off between kernel messages and the easy-to-spot LUKS passphrase prompt is left as an exercise to the user.

## Gather initial `dmesg` output
If all is well, the newly-installed Qubes OS instance should allow for user root to log in. 
Run `dmesg > dmesg.nomodeset.out` to gather an initial dmesg output.

## Gather the 'video no worky' `dmesg` output
1. Reboot and interrupt the Grub2's process, modifying the kernel parameters to no longer contain `nomodeset`.
   * If the LUKS passphrase was set, blindly enter it.
2. Wait for the system to finish booting (about 5 minutes, typically).
3. Blindly switch to a TTY via CTRL-ALT-F2.
4. Blindly log in as user root
5. Blindly run `dmesg > dmesg.out`
6. Blindly run `reboot` (this will also serve to confirm that logging in as root, and running commands blindly is possible rather than, say, the kernel having hung or some such).
   * Should this step fail, perhaps by the time step #3 was undertaken, the OS hasn't finished coming up yet. Please retry, possibly with a different TTY (say, 3 or 4 - so CTRL-ALT-F3?)

## Exfiltrate the dmesg outputs
Allow the system to boot normally, log in as user root, and sneakernet the files off the system for analysis, review, bug logging, et cetera.
