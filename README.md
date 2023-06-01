# Optimizing Linux

I am writing this guide to save my progress and let others contribute to increasing linux performance even further;
after
all, many are better than one. You can use all of them or just a few of them. **Read a topic fully before starting**.

I am currently on fedora, so some steps may vary from distro to distro.

**NOTE: This guide is not for beginners who are new to Linux** but a few of them can be used safely by them.

## Index

- [Compiling your kernel](#compiling-your-kernel)
    - [Applying patches](#applying-patches)
    - [Removing your own compiled kernel](#removing-your-own-compiled-kernel)
- [Btrfs filesystem optimizations](#btrfs-filesystem-optimizations)
- [Changing boot parameters](#changing-boot-parameters)
- [Improving boot time](#improving-boot-time)
- [Changing swappiness](#changing-swappiness)
- [Changing scaling_governor to performance](#changing-scaling_governor-to-performance)
- [Improving graphic card performance](#improving-graphic-card-performance)
- [Some other tweaks](#some-other-tweaks)

---------------------------------------------------

## Compiling your kernel

By now, everyone agrees that compiling your kernel is one of the best options to get the fastest possible speed.
You might want to google `How to make custom kernel in <distro>` to get the packages required to compile the kernel.

1. Download the [latest kernel](https://www.kernel.org/) or whatever you like. Extract it; I am going to assume a
   generic name from now on `linux-x.x.x`.


2. The next step is finding the `config` file. Most of the time, you can just run:
    ```shell
    cp -v /boot/config-$(uname -r) .config
    ```
   From inside `linux-x.x.x`, which should give an output like:
    ```shell
    '/boot/config-y.y.y-generic' -> '.config'
    ```
   if it fails, you can find config in `/proc/config.gz` or simple run `make listnewconfig` OR `make oldconfig`(it
   usually starts a long process; try finding your config in your distro source code too).


3. Edit `Makefile` and change `EXTRAVERSION` to add something. For example, "EXTRAVERSION = \<yourname>".


4. (You might wanna see the next subtopic before doing this) Now run `make xconfig`. Now a lot of optimizations are
   possible
   Here, many dead codes and modules can be removed and enabled. Let's go the safe road for now.
    - Now, one of the best things you can do is no longer build for a generic kernel. Select
        ```markdown
        -  Processor type and features
            - Processor family
                - [x] Core2/newer Xeon
        ```
      It should have been `Generic-x86-64` by default.
    - There is a lot of other stuff you can do too, but you will have to read them and see which suits
      you best. A simple way might be to just copy [clear linux config](https://github.com/clearlinux-pkgs/linux), but
      it might disable certain features(see next [Applying patches](#applying-patches)).


5. Now, you might want to run:
    ```shell
    dmesg --level=err
    dmesg --level=warn   
    ```
   To see if you can enable some extra flags for extra features. For
   example, `psmouse serio1: elantech: The touchpad can support a better bus than the old PS/2 protocol. Make sure MOUSE_PS2_ELANTECH_SMBUS and MOUSE_ELAN_I2C_SMBUS are enabled to get a better touchpad experience.`
   can be solved by enabling both of them.


6. Finally, compiling the kernel:
    ```shell
    # sed -ri '/CONFIG_SYSTEM_TRUSTED_KEYS/s/=.+/=""/g' .config
    make -j N CFLAGS='-march=native -O3 -flto -pipe' CXXFLAGS='-march=native -O3 -flto -pipe'
    make -j N CFLAGS='-march=native -O3 -flto -pipe' CXXFLAGS='-march=native -O3 -flto -pipe' modules
    sudo make modules_install
    sudo make install
    ```

   Where `N` is the number of `cores` you have, alternatively use `$(getconf _NPROCESSORS_ONLN)`.

   If any steps fail, run `make clean` and start again.


7. Making it default in grub(I am using grub2, your process might vary):
    ```shell
    sudo grub2-mkconfig -o /boot/grub2/grub.cfg
    sudo grubby --set-default /boot/vmlinuz-x.x.x-x
    ```
   You can find yours `vmlinuz-x.x.x-x` in `/boot/`

Now restart and run `uname -r` to see your kernel.

### Applying patches

There are several patches that you can use to increase performance or to make life simpler.

There are a lot of patches available, and you will have to find those that suit you best. I will be
using [graysky2](https://github.com/graysky2/kernel_gcc_patch) kernel patch here. Download the
whole [repo](https://github.com/graysky2/kernel_gcc_patch) or just the file you need. In my case, I have GCC 10 and
the latest kernel, so I will be
using [this](https://github.com/graysky2/kernel_gcc_patch/blob/master/enable_additional_cpu_optimizations_for_gcc_v10.1%2B_kernel_v5.8%2B.patch)
.

1. Copy the desired patch file into the root of the extracted linux dictionary; same place as `.config`.


2. `patch -p1 < enable_additional_cpu_optimizations_for_gcc_v10.1+_kernel_v5.8+.patch`

   You should see an output like this:

   ```shell
   patching file arch/x86/Kconfig.cpu
   patching file arch/x86/Makefile
   patching file arch/x86/Makefile_32.cpu
   patching file arch/x86/include/asm/vermagic.h
   ```


3. Now, you can start from step 4 in the previous setup and will see:
   ```markdown
   -  Processor type and features
       - Processor family
           - [x] Native optimizations autodetected by GCC
   ```

There are other patches such as [scheduling related](https://cchalpha.blogspot.com/) that you can apply to. Again try
finding your patches that suits your system.

### Removing your own compiled kernel

Try to keep the last working kernel, i.e., have a minimum of 2 kernels (the one you are using and the previous one).
**NOTE:** Removing the currently running kernel (determined by `uname -r`) will render your system
non-bootable.

1. These entries need to be removed:
   ```shell
   /boot/vmlinuz-x.x.x-x
   /boot/initrd-x.x.x-x
   /boot/System-map-x.x.x-x
   /boot/config-x.x.x-x
   /lib/modules/x.x.x-x/
   /var/lib/initramfs/x.x.x-x/
   /boot/loader/entries/*x.x.x-x
   ```


2. `sudo grub2-mkconfig -o /boot/grub2/grub.cfg` or `sudo update-grub2`

## Btrfs filesystem optimizations

1. `sudo gedit /etc/fstab`, change it to look something like this(this is on fedora, yours might vary):
    ```shell
    UUID=<do-not-change> /                       btrfs   subvol=root,x-systemd.device-timeout=0,ssd,noatime,space_cache,commit=120,compress=zstd,discard=async 0 0
    UUID=<do-not-change> /boot                   ext4    defaults        1 2
    UUID=<do-not-change>          /boot/efi               vfat    umask=0077,shortname=winnt 0 2
    UUID=<do-not-change> /home                   btrfs   subvol=home,x-systemd.device-timeout=0,ssd,noatime,space_cache,commit=120,compress=zstd,discard=async 0 0
    ```


2. `sudo systemctl daemon-reload`


3. `sudo systemctl enable fstrim.timer`

## Changing boot parameters

**Important:** I usually like disabling `mitigations`, but then again, I am on `AMD` based CPU and do not
have `Meltdown`
only `Spectre`, I do not run an unknown script, and even if I have to, I use containers and firefox with `noscript` and
a
few other security add-ons. Nonetheless, if you understand the security concerns, you can disable it and see a
substantial
boost in performance.

1. `sudo grubby --args "mitigations=off nowatchdog processor.ignore_ppc=1 amdgpu.ppfeaturemask=0xffffffff ec_sys.write_support=1" --update-kernel=ALL`

OR

1. `sudo gedit /etc/default/grub`


2. You will find a line `GRUB_CMDLINE_LINUX=" ... rhgb quiet` change it to (`...` signifies other parameters):
    ```shell
    GRUB_CMDLINE_LINUX="... rhgb quiet mitigations=off nowatchdog processor.ignore_ppc=1"
    ```

3. Also, edit `GRUB_TIMEOUT=5` to `GRUB_TIMEOUT=1.`


4. `sudo grub2-mkconfig -o /etc/grub2-efi.cfg`

   OR

   `sudo grub2-mkconfig -o /etc/grub2.cfg`

After rebooting, you can run `cat /proc/cmdline` to see your boot options.

## Improving boot time

Our last tweak kinda improved it but let's try something more.

1. Remove startup applications; I use `gnome-tweaks` for a GUI-like experience.


2. Run the following to find what service is taking the longest:

   ```shell
   systemd-analyze
   systemd-analyze blame
   systemd-analyze critical-chain
   ```

   These might vary from system to system and distro to distro; in my case(fedora), I disabled `dnf-makecache.service.`
   which took around `32s`. To do so:
    ```shell
    sudo systemctl disable NetworkManager-wait-online.service
    sudo systemctl disable dnf-makecache.service 
    sudo systemctl disable dnf-makecache.timer
    sudo gsettings set org.gnome.software download-updates false
    ```

   You might wanna google every service that you think about disabling and what it does; in my case, it just updates dnf
   cache, which I usually like to do manually.

## Changing swappiness

If you have 8GB or more ram, you might benefit from it; otherwise, leave it as it is.

1. To see current swappiness, enter `cat /proc/sys/vm/swappiness`; it should print `60`; we wanna make it 10.


2. `sudo gedit /etc/sysctl.conf`


3. Enter `vm.swappiness=10` and reboot; now step 1 should print 10.

## Changing `scaling_governor` to `performance`

Do not change it to `performance` on Ryzen based CPUs as it **_might_**(I seem to get better performance on AC, but then
again, `performance` does not seem to allow turbo boost in some cases) hurt their performance, using `ondemand`.
or `schedutil` is better(more leaning towards `schedutil` as soon as it
gets [fixed](https://www.phoronix.com/scan.php?page=article&item=linux511-amd-patch&num=1)).

1. Run `cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor` to see your current governor.


2. `echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor`

   This setting most likely will not persist during the next boot; I like to change it manually rather than making a
   systemd service(I am a laptop, and it gets hot). You might want to google how to make it persistent for your distro
   if
   you like OR:
    ```shell
    echo 'GOVERNOR="performance"' | sudo tee /etc/default/cpufrequtils
    sudo systemctl disable schedutil 
    ```

   The default is `schedutil`; you can see
   others [here](https://www.kernel.org/doc/html/v4.14/admin-guide/pm/cpufreq.html).

**Note**: You can also change the default during the kernel compilation.

## Improving graphic card performance

You can find overclocking tools specific to your GPU(s), but to make sure your graphics card isn’t being suppressed by
the OS(especially AMD):

1. Checking whether it is `auto`:

    ```shell
    cat /sys/class/drm/card0/device/power_dpm_force_performance_level
    cat /sys/class/drm/card1/device/power_dpm_force_performance_level
    ```

2. Check the parameters of GPU by:

    ```shell
    sudo cat /sys/kernel/debug/dri/0/amdgpu_pm_info
    sudo cat /sys/kernel/debug/dri/1/amdgpu_pm_info
    ```

3. Now set everything to high:

    ```shell
    sudo su
    echo high > /sys/class/drm/card0/device/power_dpm_force_performance_level
    echo high > /sys/class/drm/card1/device/power_dpm_force_performance_level
    ```

   You can change them back to `auto` if your system overheats.

## Some other tweaks

- [ArchWiki/Improving performance](https://wiki.archlinux.org/index.php/Improving_performance)

- Disabling `Cool'n'Quiet` or `speedstep` or `PowerNow!` from bios (will cause heat up on laptops, only enable it during
  gaming)

- Check other bios features, too; they vary from system to system but should have a significant boost in performance

- Using `X` instead of `Wayland` (may vary game to game)

- Using `Opengl` backend in games instead of `Vulkun` (may vary game to game)

---------------------------------------------------

## Contributing

Feel free to open an [issue](https://github.com/sn99/Optimizing-linux/issues/new)
or [editing the README](https://github.com/sn99/Optimizing-linux/edit/master/README.md) yourself.

---------------------------------------------------

## License

Licensed under either of these:

* Apache License, Version 2.0, ([LICENSE-APACHE](LICENSE-APACHE) or https://www.apache.org/licenses/LICENSE-2.0)
* MIT license ([LICENSE-MIT](LICENSE-MIT) or https://opensource.org/licenses/MIT)
