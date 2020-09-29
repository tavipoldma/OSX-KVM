### App Store problems

Do you see the "Your device or computer could not be verified" message when you
try to login to the App Store? If yes, here are the steps to fix it.

* Make sure that your wired ethernet connection is called "en0" (and not "en1" or
something else). Use "ifconfig" command to verify this.

* If the wired ethernet connection is not called "en0", then then go to Network
in System Preferences and delete all the devices, and apply the changes. Next,
delete /Library/Preferences/SystemConfiguration/NetworkInterfaces.plist file.
Finally reboot, and then use the App Store without problems.

This fix was found by Glnk2012 of https://www.tonymacx86.com/ site.

Also tweaking the `smbios.plist` file can help (?).


### Change resolution in OpenCore

```diff
diff --git a/OpenCore-Catalina/config.plist b/OpenCore-Catalina/config.plist
index 4754e8c..489570f 100644
--- a/OpenCore-Catalina/config.plist
+++ b/OpenCore-Catalina/config.plist
@@ -692,7 +692,7 @@
                        <key>ConsoleMode</key>
                        <string></string>
                        <key>Resolution</key>
-                       <string>Max</string>
+                       <string>1920x1080</string>
                        <key>ClearScreenOnModeSwitch</key>
                        <false/>
                        <key>IgnoreTextInGraphics</key>
```

### GPU passthrough notes

These steps will need to be adapted for your particular setup. A host machine
with IOMMU support is required. Consult the Arch Wiki article linked to at the
bottom of this file for exact requirements and other details.

I am running Ubuntu 17.04 on Intel i5-6500 + ASUS Z170-AR motherboard + NVIDIA
1050 Ti.

Tip: Use https://github.com/Benjamin-Dobell/nvidia-update to install nVidia
drivers on macOS.

* Enable IOMMU support on the host machine.

  Add `iommu=pt intel_iommu=on video=efifb:off` to the `GRUB_CMDLINE_LINUX_DEFAULT` line in `/etc/default/grub` file.

* Uninstall NVIDIA drivers from the host machine and blacklist the required modules.

  ```
  $ cat /etc/modprobe.d/blacklist.conf
  ... <existing stuff>

  blacklist radeon
  blacklist nouveau
  blacklist nvidia
  ```

* Enable the required kernel modules.

  ```
  # echo "vfio" >> /etc/modules
  # echo "vfio_iommu_type1" >> /etc/modules
  # echo "vfio_pci" >> /etc/modules
  # echo "vfio_virqfd" >> /etc/modules
  ```

* Isolate the passthrough PCIe devices with vfio-pci, with the help of `lspci
  -nnk` command. Adapt these commands to suit your hardware setup.

  ```
  $ lspci -nn
  ...
  01:00.0 ... NVIDIA Corporation [GeForce GTX 1050 Ti] [10de:1c82]
  01:00.1 Audio device: NVIDIA Corporation Device [10de:0fb9]
  03:00.0 USB controller: ASMedia ASM1142 USB 3.1 Host Controller [1b21:1242]
  ```

  ```
  # echo "options vfio-pci ids=10de:1c82,10de:0fb9 disable_vga=1" > /etc/modprobe.d/vfio.conf
  ```

* Update initramfs, GRUB and then reboot.

  ```
  $ sudo update-grub2
  $ sudo update-initramfs -k all -u
  ```

* Verify that the IOMMU is enabled, and vfio_pci is working as expected.
  Consult Arch Wiki again for help on this.

* On the macOS VM, install a NVIDIA Web Driver version which is appropriate for
  the macOS version. Consult http://www.macvidcards.com/drivers.html for more
  information.

  For example, macOS 10.12.5 requires version `378.05.05.15f01` whereas macOS
  10.12.6 requires version `378.05.05.25f01`.

* Boot the macOS VM using the `boot-passthrough.sh` script. At this point, the
  display connected to your passthrough PCIe device should turn on, and you
  should see the Clover boot screen. Using the keyboard, navigate to Options ->
  Graphics Injectord, and enable `Use NVIDIA Web Driver`, then boot macOS.

* Updating SMBIOS for the macOS to `iMac14,2` might be required. I did not do
  so myself.

* To reuse the keyboard and mouse devices from the host, setup "Automatic
  login" in System Preferences in macOS and configure Synergy software.

Note: Many AMD GPU devices (e.g. AMD RX 480 & RX 580) should be natively
supported in macOS High Sierra.

Note: AMD GPU devices may require configuring Clover with `Graphics > RadeonDeInit`
key enabled.


### USB passthrough notes

These steps will need to be adapted for your particular setup.

* Isolate the passthrough PCIe devices with vfio-pci, with the help of `lspci
  -nnk` command.

  ```
  $ lspci -nn
  ...
  01:00.0 ... NVIDIA Corporation [GeForce GTX 1050 Ti] [10de:1c82]
  01:00.1 Audio device: NVIDIA Corporation Device [10de:0fb9]
  03:00.0 USB controller: ASMedia ASM1142 USB 3.1 Host Controller [1b21:1242]
  ```

  Add `1b21:1242` to `/etc/modprobe.d/vfio.conf` file in the required format.

* Update initramfs, and then reboot.

  ```
  $ sudo update-initramfs -k all -u
  ```

* Use the helper scripts to isolate the USB controller.

  ```
  $ scripts/lsgroup.sh
  ### Group 7 ###
      00:1c.0 PCI bridge: Intel Corporation Sunrise ...
  ### Group 15 ###
      06:00.0 Audio device: Creative Labs Sound Core3D ...
  ### Group 5 ###
      00:17.0 SATA controller: Intel Corporation Sunrise ...
  ### Group 13 ###
      03:00.0 USB controller: ASMedia ASM1142 USB 3.1 Host Controller
  ```

  ```
  $ scripts/vfio-group.sh 13
  ```

* Add `-device vfio-pci,host=03:00.0,bus=pcie.0 \` line to the
  `boot-passthrough.sh` script.

* Boot the VM, and devices attached to the ASMedia USB controller should just
  work under macOS.


### Synergy Notes

* Get Synergy from https://sourceforge.net/projects/synergy-stable-builds.

  I installed "synergy-v1.8.8-stable-MacOSX-x86_64.dmg" on the macOS guest and
  configured it as a client.

  For automatically starting Synergy on macOS, add Synergy to "Login Items",
  System Preferences -> Users & Groups -> Select your user account -> Login Items
  -> Add a login item

* On the Linux host machine, install "synergy-v1.8.8-stable-Linux-x86_64.deb"
  or newer, configure `~/.synergy.conf` and run `synergys` command.

* The included `.synergy.conf` will need to be adapted according to your setup.


### Higher Resolution (for "UEFI + Clover")

Follow the steps below to get a higher resolution:

1. Set the desired Clover screen resolution in the relevant
   `config.plist.stripped.qemu` file and regenerate the corresponding
   `Clover*.qcow2` file (process documented in `Mojave/README.md`).

2. Ensure that the OVMF resolution is set equal to resolution set in your
   Clover.qcow2 file (default is 1024x768). This can be done via the OVMF menu,
   which you can reach with a press of the ESC button during the OVMF boot logo
   (before Clover boot screen appears). In the OVMF menu settings, set Device
   Manager -> OVMF Platform Configuration -> Change Preferred Resolution for Next
   Boot to the desired value (default is 1024x768). Commit changes and exit the
   OVMF menu.

3. Relaunch the boot script.


### Accelerated Graphics

Install VMsvga2 from [this location](https://sourceforge.net/projects/vmsvga2/). No support
is provided for this unmaintained project!

* Add `-vga vmware` to QEMU parameters in the booot script (e.g.
  boot-macOS.sh), if required.

* For Clover bootloader, add `wmv_option_fb=0x06` to the `<string>` tag of the
  `Arguments` key of the `config.plist` you use when generating the
  `CloverNG.qcow2`.

* See `UEFI/README.md` for GPU passthrough notes.

* Note: There is no working QXL driver for macOS so far.


### Virtual Sound Device

No support is provided for this. You are on your own. The sound output is known
to be choppy and distorted.

* Add `-device ich9-intel-hda -device hda-duplex` to the VM configuration.
  `boot-macOS.sh` already has this change.

* To get sound on your virtual Mac, install the VoodooHDA driver from
  [here](https://sourceforge.net/projects/voodoohda/files/).

Note: It seems that playback of Flash videos requires an audio device to be
present.


### Building QEMU from source

See http://wiki.qemu-project.org/Hosts/Linux for help.

```
$ git clone https://github.com/kholia/qemu.git

$ cd qemu

$ git checkout macOS

$ ./configure --prefix=/home/$(whoami)/QEMU --target-list=x86_64-softmmu --audio-drv-list=pa

$ make clean; make; make install
```


### Connect iPhone / iPad to macOS guest

Some folks are using https://www.virtualhere.com/ to connect iPhone / iPad to
the macOS guest.

Update: It appears that VirtualHere doesn't work on modern macOS versions.

Please passthrough a PCIe USB card to the virtual machine to be able to connect
iDevices to it.


### Exposing AES-NI instructions to macOS

Add `+aes` argument to the `-cpu` option in `boot-macOS.sh` file.

``` diff
diff --git a/boot-macOS.sh b/boot-macOS.sh
index 5948b8a..3acc123 100755
--- a/boot-macOS.sh
+++ b/boot-macOS.sh
@@ -18,7 +18,7 @@
 # Use "-device usb-tablet" instead of "-device usb-mouse" for better mouse
 # behaviour. This requires QEMU >= 2.9.0.

-qemu-system-x86_64 -enable-kvm -m 3072 -cpu Penryn,kvm=off,vendor=GenuineIntel \
+qemu-system-x86_64 -enable-kvm -m 3072 -cpu Penryn,kvm=off,vendor=GenuineIntel,+aes \
          -machine pc-q35-2.4 \
          -smp 4,cores=2 \
          -usb -device usb-kbd -device usb-mouse \
```

Other host CPU features can be similarly exposed to the macOS guest.

The following command can be used on macOS to verify that AES-NI instructions are exposed,

```
sysctl -a | grep machdep.features
```

On machines with OpenSSL installed, the following two commands can be used to
check AES-NI performance,

```
openssl speed aes-128-cbc

openssl speed -evp aes-128-cbc  # uses AES-NI
```


### Exposing AVX and AVX2 instructions to macOS

Exposing AVX and AVX2 instructions to macOS requires support for these
instructions on the host CPU.

The `boot-clover.sh` script already exposes AVX and AVX2 instructions to the
macOS guest by default. Modify or comment out the `MY_OPTIONS` line in
`boot-clover.sh` file in case you are having problems.

To enable AVX2, do the following change,

`Clover boot menu -> Options -> Binaries patching -> Fake CPUID -> 0x0306C0  # for Haswell`

For details, see [this wiki](https://clover-wiki.zetam.org/Configuration/KernelAndKextPatches) page.

Once enabled, the following commands can be used to confirm the presence of AVX
and AVX2 instructions on the macOS guest.

```
$ sysctl -a | grep avx
hw.optional.avx2_0: 1
hw.optional.avx1_0: 1

$ sysctl -a | grep leaf7
machdep.cpu.leaf7_features: SMEP BMI1 AVX2 BMI2
machdep.cpu.leaf7_feature_bits: 424
```


### Running Docker for Mac

Docker for Mac requires enabling nested virtualization on your host machine,

```
modprobe -r kvm_intel
modprobe kvm_intel nested=1
```

Also you have to add `vmx,rdtscp` arguments to the `-cpu` option in
`boot-macOS.sh` file.


### Using virtio-net-osx with macOS

Configuration options for macOS Sierra (thanks to virtio-net-osx project users),

```
-netdev user,id=hub0port0 \
-device virtio-net,netdev=hub0port0,id=eth0 \
-set device.eth0.vectors=0
```

Adapt these to your use case. These changes need to be made in the `boot-*`
scripts. On the guest, install the included `Virtio-Net-Driver-0.9.4.pkg`
package.

Update: This is no longer recommended. Use `vmxnet3` adapter instead.


### Using virtio-blk-pci with macOS

Newer macOS (namely Mojave+) have support for some virtio drivers.

This can be enabled by applying the following change to `boot-macOS-NG.sh` to
get some performance gain.

```diff
-         -device ide-hd,bus=sata.4,drive=MacHDD \
+         -device virtio-blk-pci,drive=MacHDD \
```


### Permission problems with libvirt / qemu?

```
sudo setfacl -m u:libvirt-qemu:rx <path>  # fix virt-manager perm problems
```


### Extract .pkg files

* http://mackyle.github.io/xar/ is unmaintained and fails for many `.pkg` files.

* Use a modern version of `7-Zip` instead.

  ```
  7z l example.pkg

  7z x example.pkg

  gunzip -c <something>.pkg/Payload | cpio -i
  ```
