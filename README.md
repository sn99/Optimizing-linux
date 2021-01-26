# Optimizing Linux

I am writing this guide as to save my progress and let others contribute to increase the performance even further, after all many are better than one. You can use all of them or just a few of them.

I am currently on fedora so some steps may vary from distro to distro.


<details><summary>Index</summary>
<p>

- [Compiling your own kernel](#compiling-your-own-kernel)
  - [Removing your own compiled kernel](#removing-your-own-compiled-kernel)
- [Btrfs filesystem optimizations](#btrfs-filesystem-optimizations)
- [Changing boot parameters](#changing-boot-parameters)
- [Improving boot time](#improving-boot-time)
- [Changing swappiness](#changing-swappiness)
- [Changing scaling_governor to performance](#changing-scaling_governor-to-performance)
- [Improving graphic card performance](#improving-graphic-card-performance)
- [Some other tweaks](#some-other-tweaks)

</p>
</details>

---------------------------------------------------

## Compiling your own kernel

I think by now everyone agrees that compiling your own kernel is one of the best options to get fastest possible speed.
You might wanna google `How to make custom kernel in <distro>` to get the packages required to compile the kernel.

1. Download the [latest kernel](https://www.kernel.org/) or whatever you might like.
Extract it, I am gonna assume a generic name from now on `linux-x.x.x`.


2. The next step is finding the `config` file. Most of the time you can just run:
    ```shell
    cp -v /boot/config-$(uname -r) .config
    ```
    from inside `linux-x.x.x` which should give an output like:
    ```shell
    '/boot/config-y.y.y-generic' -> '.config'
    ```
    if it fails you can find config in `/proc/config.gz` or simple run `make oldconfig`(it usually starts a long process, try finding your config in your distro source code too). 


3. Edit `Makefile` and change `EXTRAVERSION` to add something. For example, "EXTRAVERSION = \<yourname>".


4. Now run `make xconfig`. Now a lot of optimizations are possible here and a lot of dead code and modules can be removed and enabled. Let's go the safe road for now.
    - Now one of the best thing you can do is no longer build for a generic kernel. Select
        ```markdown
        -  Processor type and features
            - Processor family
                - [x] Core2/newer Xeon
        ```
        It should have been `Generic-x86-64` by default.
    - There are a lot of other stuff you can do too but then you will have to yourself read them and see which suits best. A simple way might be to just copy [clear linux config](https://github.com/clearlinux-pkgs/linux) but it might disable certain features(see next).
    
    
5. Now you might wanna run: 
    ```shell
    dmesg --level=err
    dmesg --level=warr    
    ```
   to see if you can enable some extra flags for extra features. For example `psmouse serio1: elantech: The touchpad can support a better bus than the too old PS/2 protocol. Make sure MOUSE_PS2_ELANTECH_SMBUS and MOUSE_ELAN_I2C_SMBUS are enabled to get a better touchpad experience.` can be solved by enabling both of them.


6. Finally compiling the kernel:
    ```shell
    sed -ri '/CONFIG_SYSTEM_TRUSTED_KEYS/s/=.+/=""/g' .config
    make -j N "KCFLAGS=-g -O3 -march=native -pipe"
    make -j N "KCFLAGS=-g -O3 -march=native -pipe" modules
    make -j N "KCFLAGS=-g -O3 -march=native -pipe" modules_install
    make -j N "KCFLAGS=-g -O3 -march=native -pipe" install
    ```
   
    Where `N` is the number of `cores` you have, alternatively use `$(getconf _NPROCESSORS_ONLN)`. 
    
    If any of the step fails run `make clean` and start again.


7. Making it default in grub(I am using grub2, your process might vary):
    ```shell
    sudo grub2-mkconfig -o /boot/grub2/grub.cfg
    sudo grubby --set-default /boot/vmlinuz-x.x.x-x
    ```
   You can find yours `vmlinuz-x.x.x-x` in `/boot/`

Now restart and run `uname -r` to see your own kernel.

### Removing your own compiled kernel

Try to keep the last working kernel i.e. have a minimum of 2 kernels (the one you are using and the previous one).

1. These entries need to be removed: 
   ```shell
   /boot/vmlinuz-x.x.x-x
   /boot/initrd-x.x.x-x
   /boot/System-map-x.x.x-x
   /boot/config-x.x.x-x
   /lib/modules/x.x.x-x/
   /var/lib/initramfs/x.x.x-x/
   ```

2. `sudo grub2-mkconfig -o /boot/grub2/grub.cfg` or `sudo update-grub2`

## Btrfs filesystem optimizations

1. `sudo gedit /etc/fstab`, change it ot look something like this(this is on fedora, yours might vary):
    ```shell
    UUID=<do-not-change> /                       btrfs   subvol=root,x-systemd.device-timeout=0,ssd,noatime,space_cache,commit=120,compress=zstd,discard=async 0 0
    UUID=<do-not-change> /boot                   ext4    defaults        1 2
    UUID=<do-not-change>          /boot/efi               vfat    umask=0077,shortname=winnt 0 2
    UUID=<do-not-change> /home                   btrfs   subvol=home,x-systemd.device-timeout=0,ssd,noatime,space_cache,commit=120,compress=zstd,discard=async 0 0
    ```
   

2. `sudo systemctl daemon-reload`


3. `sudo systemctl enable fstrim.timer`


## Changing boot parameters

**Important:** I usually like disabling `mitigations`, but them again I am on `AMD` based cpu and do not have `Meltdown` only `Spectre`, I do not run unknown script and even if I have to I use containers and use firefox with `noscript` and a few other security addons. Nonetheless if you understand the security concerns you can disable it and see a substantial boost in performance.

1. `sudo gedit /etc/default/grub`


2. You will find a line `GRUB_CMDLINE_LINUX=" ... rhgb quiet` change it to (`...` signifies other parameters):
    ```shell
    GRUB_CMDLINE_LINUX="... rhgb quiet mitigations=off nowatchdog processor.ignore_ppc=1"
    ```
   
3. Also edit `GRUB_TIMEOUT=5` to `GRUB_TIMEOUT=1`


4. `sudo grub2-mkconfig -o /etc/grub2-efi.cfg` 
   
    OR
   
   `sudo grub2-mkconfig -o /etc/grub2.cfg`

After rebooting you can run `cat /proc/cmdline` to see your boot options.


## Improving boot time

Our last tweak kinda improved it but let's try something more.

1. Remove startup applications, I use `gnome-tweaks` for a GUI like experience.


2. Run the following to find what service is taking the longest:
   
   ```shell
   systemd-analyze
   systemd-analyze blame
   systemd-analyze critical-chain
   ```

    This might vary from system to system and distro to distro, in my case(fedora) I disabled `dnf-makecache.service` which took around `32s`. To do so:
    ```shell
    systemctl disable dnf-makecache.service 
    systemctl disable dnf-makecache.timer
    gsettings set org.gnome.software download-updates false
    ```
   
    You might wanna google every service that you think about disabling and what it does, in my case it just updates dnf cache which I usually like to do manually.


## Changing swappiness

If you have 8GB or more ram you might benefit from it otherwise leave it as it is.

1. To see current swappiness enter `cat /proc/sys/vm/swappiness`, it should print `60`, we wanna make it 10.


2. `sudo gedit /etc/sysctl.conf`


3. Enter `vm.swappiness=10` and reboot, now step 1 should print 10.


## Changing `scaling_governor` to `performance`

1. Run `cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor` to see your current governor.


2. `echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor`

    This setting most likely will not persist during the next boot, I like to change it manually rather than making a systemd service(I am an laptop and it gets hot). You might wanna google how to make it persistent for your distro if you like OR:
    ```shell
    echo 'GOVERNOR="performance"' | sudo tee /etc/default/cpufrequtils
    sudo systemctl disable schedutil 
    ```
   
    The default is `schedutil`, you can see others [here](https://www.kernel.org/doc/html/v4.14/admin-guide/pm/cpufreq.html).

    
Note: You can also change the default during compilation of the kernel.
   

## Improving graphic card performance 

You can find overclocking tools specific to you GPU(s), but to make sure your graphics card isn’t being suppressed by the OS(especially AMD):

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
    echo auto > /sys/class/drm/card0/device/power_dpm_force_performance_level
    echo auto > /sys/class/drm/card1/device/power_dpm_force_performance_level
    ```
   
    You can change them back to `auto` if your system overheats.
    

## Some other tweaks

- Disabling `Cool'n'Quiet` or `speedstep` or `PowerNow!` from bios (will cause heatup on laptops)

- Using `X` instead of `Wayland` (may vary game to game)
  
- Using 'Opengl' backend in games instead of `Vulkun` (may vary game to game)


---------------------------------------------------

## License

Licensed under

 * MIT license ([LICENSE](LICENSE) or http://opensource.org/licenses/MIT)
