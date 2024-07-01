External USB fuzzing for Linux kernel
=====================================

Syzkaller supports fuzzing the Linux kernel USB subsystem externally (as can be done by plugging in a programmable USB device like [Facedancer](https://github.com/usb-tools/Facedancer)). This allowed finding over [300 bugs](/docs/linux/found_bugs_usb.md) in the Linux kernel USB stack so far.

USB fuzzing support consists of 3 parts:

1. Syzkaller changes; see the [Internals](/docs/linux/external_fuzzing_usb.md#Internals) section for details.
2. Kernel interface for USB device emulation called [Raw Gadget](https://github.com/xairy/raw-gadget), which is now in the mainline kernel.
3. KCOV changes that allow to collect coverage from background kernel threads and interrupts, which are now in the mainline kernel.

See the OffensiveCon 2019 [Coverage-Guided USB Fuzzing with Syzkaller](https://docs.google.com/presentation/d/1z-giB9kom17Lk21YEjmceiNUVYeI6yIaG5_gZ3vKC-M/edit?usp=sharing) talk ([video](https://www.youtube.com/watch?v=1MD5JV6LfxA)) for some (partially outdated) details.

As USB fuzzing requires kernel side support, for non-mainline kernels you need all mainline patches that touch `drivers/usb/gadget/udc/dummy_hcd.c`, `drivers/usb/gadget/legacy/raw_gadget.c` and `kernel/kcov.c`.


## Internals

Currently, syzkaller defines 6 USB pseudo-syscalls (see [syzlang descriptions](/sys/linux/vusb.txt) and [pseudo-syscalls](/executor/common_usb.h) [implementation](/executor/common_usb_linux.h), which relies on the Raw Gadget interface linked above):

1. `syz_usb_connect` - connects a USB device. Handles all requests to the control endpoint until a `SET_CONFIGURATION` request is received.
2. `syz_usb_connect_ath9k` - connects an `ath9k` USB device. Compared to `syz_usb_connect`, this syscall also handles firmware download requests that happen after `SET_CONFIGURATION` for the `ath9k` driver.
3. `syz_usb_disconnect` - disconnects a USB device.
4. `syz_usb_control_io` - sends or receives a control message over endpoint 0.
5. `syz_usb_ep_write` - sends a message to a non-control endpoint.
6. `syz_usb_ep_read` - receives a message from a non-control endpoint.

These pseudo-syscalls targeted at a few different layers:

1. USB core enumeration process is targeted by the generic `syz_usb_connect` variant. As the USB device descriptor fields for this pseudo-syscall get [patched](/sys/linux/init_vusb.go) by syzkaller runtime, `syz_usb_connect` also briefly targets the enumeration process of various USB drivers.
2. Enumeration process for class-specific drivers is targeted by `syz_usb_connect$hid`, `syz_usb_connect$cdc_ecm`, etc. (the device descriptors provided to them have fixed identifying USB IDs to always match to the same USB class driver) accompanied by matching `syz_usb_control_io$*` pseudo-syscalls.
3. Subsequent communication through non-control endpoints for class-specific drivers is not targeted by existing descriptions yet for any of the supported classes. But it can be triggered through generic `syz_usb_ep_write` and `syz_usb_ep_read` pseudo-syscalls.
4. Enumeration process for device-specific drivers is not covered by existing descriptions yet.
5. Subsequent communication through non-control endpoints for device-specific drivers is partially described only for `ath9k` driver via `syz_usb_connect_ath9k`, `syz_usb_ep_write$ath9k_ep1` and `syz_usb_ep_write$ath9k_ep2` pseudo-syscalls.

There are [runtests](/sys/linux/test/) for USB pseudo-syscalls. They are named starting with the `vusb` prefix and can be run with:

```
./bin/syz-runtest -config=usb-manager.cfg -tests=vusb
```


## Things to improve

The core support for USB fuzzing is in place, but there's still a place for improvements:

1. Remove the device from `usb_devices` in `syz_usb_disconnect` and don't call `lookup_usb_index` multiple times within `syz_usb_connect`. Currently, this causes some reproducers to have the `repeat` flag set when it's not required.

2. Add descriptions for more relevant USB classes and drivers.

3. Resolve TODOs from [sys/linux/vusb.txt](/sys/linux/vusb.txt).

4. Implement a proper way for dynamically extracting relevant USB ids from the kernel (a related [discussion](https://www.spinics.net/lists/linux-usb/msg187915.html)).

5. Add a mode for standalone fuzzing of physical USB hosts (by using Raspberry Pi Zero, see below).
This includes at least: a. making sure that current USB emulation implementation works properly on different OSes (there are some [differences](https://github.com/RoganDawes/LOGITacker/blob/USB_host_enum/fingerprint_os.md#derive-the-os-from-the-fingerprint) in protocol implementation);
b. using USB requests coming from the host as a signal (like coverage) to enable "signal-driven" fuzzing,
c. making UDC driver name configurable for `syz-execprog` and `syz-prog2c`.

6. Generate syzkaller programs from usbmon trace that is produced by actual USB devices (this should make the fuzzer to go significantly deeper into the USB drivers code).


## Setting up

1. Make sure the version of the kernel you're using is at least 5.7. It's recommended to backport all kernel patches that touch kcov, USB Raw Gadget, and USB Dummy UDC/HCD.

2. Configure the kernel: at the very least, `CONFIG_USB_RAW_GADGET=y` and `CONFIG_USB_DUMMY_HCD=y` must be enabled.

    The easiest option is to use the [config](/dashboard/config/linux/upstream-usb.config) from the syzbot USB fuzzing instance.

3. Build the kernel.

4. Optionally update syzkaller descriptions by extracting USB IDs using the [instructions](/docs/linux/external_fuzzing_usb.md#updating-syzkaller-usb-ids) below.

5. Enable `syz_usb_connect`, `syz_usb_disconnect`, `syz_usb_control_io`, `syz_usb_ep_write` and `syz_usb_ep_read` pseudo-syscalls in the manager config.

6. Set `sandbox` to `none` in the manager config.

7. Pass `dummy_hcd.num=8` (or whatever number you use for `procs`) to the kernel command line in the manager config.

8. Run.


## Updating syzkaller USB IDs

Syzkaller uses a list of hardcoded [USB IDs](/sys/linux/init_vusb_ids.go) that are [patched](/sys/linux/init_vusb.go) into `syz_usb_connect` by syzkaller runtime. One of the ways to make syzkaller target only particular USB drivers is to alter that list. The instructions below describe a hackish way to generate syzkaller USB IDs for all USB drivers enabled in your `.config`.

1. Apply [this](/tools/syz-usbgen/usb_ids.patch) kernel patch.

2. Build and boot the kernel.

3. Connect a USB HID device. In case you're using a `CONFIG_USB_RAW_GADGET=y` kernel, use the
[keyboard emulation program](https://raw.githubusercontent.com/xairy/raw-gadget/master/examples/keyboard.c).

4. Use [syz-usbgen](/tools/syz-usbgen/usbgen.go) script to update [syzkaller descriptions](/sys/linux/init_vusb_ids.go):

    ```
    ./bin/syz-usbgen $KERNEL_LOG ./sys/linux/init_vusb_ids.go
    ```

5. Don't forget to revert the applied patch and rebuild the kernel before doing actual fuzzing.


## Running reproducers with Raspberry Pi Zero W

It's possible to run syzkaller USB reproducers by using a Linux board plugged into a physical USB host.

These instructions describe how to set this up on a Raspberry Pi Zero W, but any other board that has a working USB UDC driver can be used as well.

1. Download the latest `Raspberry Pi OS with desktop` image from [here](https://www.raspberrypi.com/software/operating-systems/).

2. Extract and flash the downloaded image into an SD card:

    ``` bash
    unxz 2024-03-15-raspios-bookworm-armhf.img.xz
    sudo dd if=2024-03-15-raspios-bookworm-armhf.img of=/dev/sdb bs=4M conv=fsync
    ```

3. [Create](https://forums.raspberrypi.com/viewtopic.php?t=333248&p=1994926#p1994926) a user `pi` with the password `raspberry` by creating a `bootfs/userconf.txt` file with the following contents:

   ```
   pi:$6$c70VpvPsVNCG0YR5$l5vWWLsLko9Kj65gcQ8qvMkuOoRkEagI90qi3F/Y7rm8eNYZHW8CY6BOIKwMH7a3YYzZYL90zf304cAHLFaZE0
   ```

4. [Enable UART](https://www.raspberrypi.com/documentation/computers/config_txt.html#enable_uart) by appending `enable_uart=1` to the end of the `bootfs/config.txt` file.

5. Boot the board and get a shell over UART as described [here](https://learn.adafruit.com/raspberry-pi-zero-creation/give-it-life).

    For this step, you will need a USB-UART adapter.

    Note that getting the login prompt over UART takes a few minutes when you boot the board for the first time.

6. Get the board connected to the internet (plug in a USB Ethernet adapter or use Wi-Fi).

7. Update: `sudo apt-get update && sudo apt-get dist-upgrade && sudo rpi-update && sudo reboot`.

8. Install useful packages: `sudo apt-get install vim git`.

9. [Install](https://www.raspberrypi.com/documentation/computers/linux_kernel.html#kernel-headers) Linux kernel headers:

   ``` bash
   sudo apt install linux-headers-rpi-{v6,v7,v7l}
   ```

10. Setup the dwc2 USB gadget driver:

    ``` bash
    echo "dtoverlay=dwc2" | sudo tee -a /boot/firmware/config.txt
    echo "dwc2" | sudo tee -a /etc/modules
    sudo reboot
    ```
11. Clone the Raw Gadget [repository](https://github.com/xairy/raw-gadget).

12. Build and load the USB Raw Gadget module following [this](https://github.com/xairy/raw-gadget/tree/master/raw_gadget).

    Depending on the used kernel version, you might need undo the revert `patches/usb_gadget_probe_driver.patch` to build the module.

13. Connect the Raspberry Pi USB OTG connector (titled `USB` on the board) to a USB host.

    You will need to unplug the USB Ethernet adapter for this step if you have it plugged in.
    You will also need to replug it back in for any further steps that require downloading things.

14. Build and run the Raw Gadget [keyboard emulator program](https://github.com/xairy/raw-gadget/tree/master/examples):

    ``` bash
    cd raw-gadget/examples
    make
    sudo ./keyboard 20980000.usb 20980000.usb
    ```

15. Make sure that you see the letter `x` being entered on the host.

    With this step, you confirmed that Raw Gadget is working properly.

16. On host, download syzkaller and apply the patch below:

    ``` bash
    git clone https://github.com/google/syzkaller
    cd syzkaller
    # Put the patch below into /tmp/syzkaller.patch
    git apply /tmp/syzkaller.patch
    ```

    Patch:

    ``` c
    diff --git a/executor/common_usb_linux.h b/executor/common_usb_linux.h
    index b706663f8..f06fd5e3b 100644
    --- a/executor/common_usb_linux.h
    +++ b/executor/common_usb_linux.h
    @@ -303,9 +303,7 @@ static volatile long syz_usb_connect_impl(uint64 speed, uint64 dev_len, const ch
     
            // TODO: consider creating two dummy_udc's per proc to increace the chance of
            // triggering interaction between multiple USB devices within the same program.
    -       char device[32];
    -       sprintf(&device[0], "dummy_udc.%llu", procid);
    -       int rv = usb_raw_init(fd, speed, "dummy_udc", &device[0]);
    +       int rv = usb_raw_init(fd, speed, "20980000.usb", "20980000.usb");
            if (rv < 0) {
                    debug("syz_usb_connect: usb_raw_init failed with %d\n", rv);
                    return rv;
    ```

17. Cross-compile `syz-execprog` and `syz-executor`:

    ``` bash
    # Update '...' below.
    alias syz-env=".../syzkaller/tools/syz-env"
    syz-env make generate
    syz-env GOARM=5 make TARGETARCH=arm execprog
    syz-env make TARGETARCH=arm executor
    ```

    Note that `syz-env` requires Docker installed and takes a while to set up a container when called for the first time.

18. Copy `./bin/linux_arm/syz-execprog` and `./bin/linux_arm/syz-executor` onto Raspberry Pi.

    One option to copy the files is to enable the SSH server on the Raspbery Pi [via](https://www.raspberrypi.com/documentation/computers/remote-access.html#ssh) `sudo raspi-config`.

    Another is to copy the files onto the SD card.

19. On Raspberry Pi, check that you can execute a simple syzkaller program:

    ``` bash
    $ cat socket.log
    r0 = socket$inet_tcp(0x2, 0x1, 0x0)
    $ sudo ./syz-execprog -executor ./syz-executor -threaded=0 -collide=0 -procs=1 -enable='' -debug socket.log
    ```

    Make sure that you see something like:

    ```
    #0 [134ms] -> socket$inet_tcp(0x2, 0x1, 0x0)
    #0 [134ms] <- socket$inet_tcp=0x3
    ```

20. You should now be able to execute syzkaller USB programs:

    ``` bash
    $ cat usb.log
    r0 = syz_usb_connect(0x0, 0x24, &(0x7f00000001c0)={{0x12, 0x1, 0x0, 0x8e, 0x32, 0xf7, 0x20, 0xaf0, 0xd257, 0x4e87, 0x0, 0x0, 0x0, 0x1, [{{0x9, 0x2, 0x12, 0x1, 0x0, 0x0, 0x0, 0x0, [{{0x9, 0x4, 0xf, 0x0, 0x0, 0xff, 0xa5, 0x2c}}]}}]}}, 0x0)
    $ sudo ./syz-execprog -executor ./syz-executor -slowdown=3 -threaded=0 -collide=0 -procs=1 -enable='' -debug usb.log
    ```

    The `slowdown` parameter is a scaling factor which can be used for increasing the syscall timeouts.

21. The following steps optional. You may use a UART console and a normal USB cable instead of SSH and Zero Stem.

22. Set up a Wi-Fi hotspot.

23. Enable SSH if you haven't yet.

24. Optionally solder [Zero Stem](https://zerostem.io/) onto your Raspberry Pi Zero W.

25. You can now connect the board to an arbitrary USB port, wait for it to boot, join its Wi-Fi network, `ssh` onto it, and run arbitrary syzkaller USB programs.
