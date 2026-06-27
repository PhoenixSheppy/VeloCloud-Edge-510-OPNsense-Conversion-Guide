# VeloCloud-Edge-510-OPNsense-Conversion-Guide
These routers are [cheaper than dirt](https://www.ebay.com/itm/297261007996) on eBay (sub-$20/unit!), and they're just x86 boxes disguised as fancy cloud-managed routers, so let's repurpose them!

## Why do we need to do this?
VeloCloud (VMware, Dell, VMware??) decided in their infinite wisdom to configure this device so that if you try any sketchy stuff, like running an unsupported (unauthorized) OS it won't work.

By default, it'll reboot after 60 seconds if it doesn't receive a response from a 'watchdog program' that's included with the factory OS.

Some dudes probably way older than I am figured out how to get past that by performing some shinnanigans outlined in these instructions. 

I took everything in [this forum post](https://forum.netgate.com/topic/142371/what-is-this-board-help/1), and condensed it down (removed the unrelated stuff) so that we can convert these devices to literally anything else but VeloCloud OS.

### Requirements:
You will need the following items:

* A "Velocloud SD-wan Edge 510 Managed Wireless Appliance" (Non-wireless models work as well, you'll be removing the wireless card anyway to complete this process!)
* A USB-Flash drive, preferably 8GB or larger (This is to store the installer for OPNSense)
* A Mini-PCIe to PCIe Adapter Kit, like [this one](https://www.amazon.com/dp/B0C2WJXQBF)
* A GPU (cheap-o will work, I used a GT-710, any PCIe GPU made in the last 10 years should work)
* An ATX PSU (This is required to power the PCIe-board in the adapter kit)
* A Screwdriver kit, including a star-bit set (Literally just get an iFixit Kit)
* A Mini-USB Cable (This is to access the serial interface hidden under the cover)
* A will to live (This is gonna suck lol)

## Step 1: Download necessary items:

To begin, you'll need to download a few programs:

* OPNSense AMD64-VGA Installer: https://opnsense.org/download/
* BalenaEtcher: https://etcher.balena.io/
* Putty / terminal emulation software: https://www.chiark.greenend.org.uk/~sgtatham/putty/

## Step 2: Disassembly:

# UPDATE: YOU CAN SKIP THIS STEP NOW UNLESS YOU ABSOLUTELY NEED A SCREEN!!! JUMP TO STEP 5!!!

Start by removing the serial-port / SIM-card slot cover on the side with the ethernet ports, there should be 1 screw and it should come right off, exposing the mini-USB port. You should be able to plug that into a PC, and connect at 115200 Baud.

Then, remove all 4 feet, and the 4 star-bit screws holding the shell to the top of the device.

<img src="https://raw.githubusercontent.com/PhoenixSheppy/VeloCloud-Edge-510-OPNsense-Conversion-Guide/refs/heads/main/pics/back_case.jpg" height="250">

Now, remove the top shell from the device, exposing the mainboard.

<img src="https://raw.githubusercontent.com/PhoenixSheppy/VeloCloud-Edge-510-OPNsense-Conversion-Guide/refs/heads/main/pics/remove_cover.jpg" height="250">

## Step 3: (If applicable) Remove the wireless card:

Remove the wireless card by unplugging the antennas, and removing the (1) screw holding it down (SAVE THIS!) and put it aside.

<img src="https://raw.githubusercontent.com/PhoenixSheppy/VeloCloud-Edge-510-OPNsense-Conversion-Guide/refs/heads/main/pics/remove_wireless_card.jpg" height="250">

*If you're like me, and don't plan on using the wireless at all, rip those antennas out and toss them in the trash.*

## Step 4: Install the Mini-PCIe Adapter Kit:

Start by installing the adapter-card where the wireless card was inserted, and affix it to the board using the screw you saved from earlier.

<img src="https://raw.githubusercontent.com/PhoenixSheppy/VeloCloud-Edge-510-OPNsense-Conversion-Guide/refs/heads/main/pics/install_mini_pcie_adapter.jpg" height="250">

Then, plug the included cable into the card and the other end into the external PCIe board.

<img src="https://raw.githubusercontent.com/PhoenixSheppy/VeloCloud-Edge-510-OPNsense-Conversion-Guide/refs/heads/main/pics/install_cable_pcie_adapter.jpg" height="250">

Next, plug the included power cable 6-pin into the PCIe board, and the other end into the SATA power cable on your ATX PSU.

<img src="https://raw.githubusercontent.com/PhoenixSheppy/VeloCloud-Edge-510-OPNsense-Conversion-Guide/refs/heads/main/pics/install_power_pcie_adapter.jpg" height="250">

Finally, seat your GPU into the external PCIe board, and plug in your power supply to the wall (DO NOT TURN IT ON YET)

<img src="https://raw.githubusercontent.com/PhoenixSheppy/VeloCloud-Edge-510-OPNsense-Conversion-Guide/refs/heads/main/pics/install_gpu_pcie_adapter.jpg" height="250">

(Obviously, plug the GPU into your monitor using whatever cable works)

<img src="https://raw.githubusercontent.com/PhoenixSheppy/VeloCloud-Edge-510-OPNsense-Conversion-Guide/refs/heads/main/pics/install_hdmi_pcie_adapter.jpg" height="250">

## Step 5: Let's flash it!

Now, plug your mini-USB cable into the device, and the other end to your PC. Using a program like Tabby.sh (My personal preference, ymmv) or Putty, connect at 115200 Baud, and plug in the device, you should see some output as it boots up. Allow it to boot into VeloCloud OS.

The default login is `root | VeloHelloXXX` (XXX=last-3 of S/N on bottom of device)

Once logged in, navigate to /root by running `cd /root`

From there, let's go ahead and network this bad-boy. Plug in an ethernet cable from your router, switch, or whatever to the 4th port (GE/4) and allow it to establish a link and connect.

Make a directory in /root, for simplicity, I'm going to name it "illegal-firmware" by running `mkdir illegal-firmware` because that sounds bad-ass.

Next, cd into that directory we just created `cd illegal-firmware`

Then, grab a copy of that BIOS from earlier, by running `wget https://raw.githubusercontent.com/PhoenixSheppy/VeloCloud-Edge-510-OPNsense-Conversion-Guide/refs/heads/main/firmware/2017-4-10-coreboot.rom` and allow it to download to your new directory.

Pop back out to the root directory `cd ..`

Next, flash the DMI (I don't know what this is, but it's all over the netgate forum, so we're doing it) using this command: `./dmi-tool -u illegal-firmware/2017-4-10-coreboot.rom` 

Follow that with this command: `./dmi-tool -w -p EDGE510 -v 1`

Finally, flash that firmware by running `flashrom --programmer internal --write illegal-firmware/2017-4-10-coreboot.rom` and let it install. Once finished, proceed.

**FINISH HIM!!!** - to finalize getting rid of that *pesky watchdog* timer that prohibits us from running whatever we want, run the following, in order:
`i2cset -y 1 0x24 0x00 0x00`
`i2cset -y 1 0x24 0x01 0x00`

After you've run those two commands, your device is **FREE** from VMware/VeloCloud's *shackles*! *You can run whatever you want on it*! 

Windows? **HELL YEAH!** 

MacOS? *Man you must really hate yourself...* 

OPNSense? ***THAT'S WHY YOU'RE HERE!***

## Step 6: Install OPNsense:

Now, plug in your USB-drive with the OPNSense installer, and plug your device back in. It should begin to boot. (Fun-fact, the device prioritizes USB-Boot first, so no intervention is required!)

(You'll know if step 5 worked if you see a line that says: `Disabling Watchdog timer... done.`)

Once it boots, it should boot into the OPNsense installer, you will lose view of the installer as it's a VGA installer, this is OK. We're going to SSH into the device with your computer.

Plug your 'internet connection' into GE2 and your computer's ethernet into GE1 (I don't know why, but OPNsense flips the WAN/LAN ports, you can it it straight later.)

Your device should automatically get a DHCP address from OPNsense once it boots up. If it doesn't, something is wrong, check your install media and start over from the beginning of this step.

SSH into the device using the credentials `installer | opnsense` and follow the on-screen directions, picking a language, and keyboard type.

Then, install using UFS (ZFS seems silly here since we only have 1 physical storage device, the onboard flash), and select the onboard flash as your target.

Allow it to install, then change your root password (please lord god do this lol) and shutdown.

*presto!* You've turned this literal e-waste router into a new-fancy router for the modern age!

**Look at you, superstar!**

<img src="https://raw.githubusercontent.com/PhoenixSheppy/VeloCloud-Edge-510-OPNsense-Conversion-Guide/refs/heads/main/pics/superstar.jpg" height="250">

## Step 7: Headless Serial Console Setup (No GPU Required)

After installing OPNsense, you can manage the device entirely over serial — no GPU, no HDMI, no monitor needed. However, there are a few OPNsense quirks that need fixing before serial login works properly.

### Background: How the serial port works on this hardware

The mini-USB port under the side cover exposes **two hardware UARTs** via an internal USB-to-serial chip:

- **COM1 (0x3F8)** — FreeBSD calls this `ttyu0`, and it's where the FreeBSD kernel sends its console output by default
- **COM2 (0x2F8)** — FreeBSD calls this `ttyu1`, and this is the port that coreboot and the FreeBSD bootloader use for their output

This means without any fixes, you'd see bootloader/module-loading output (from COM2) but then silence during OPNsense startup (which goes to COM1), followed by a login prompt on COM2. You'd also be unable to log in due to a PAM configuration issue.

The fixes below address all of this and make everything persist across reboots.

### Fix 1: Full boot output on serial

Add `comconsole_port="0x2f8"` to `/boot/loader.conf.local` to tell the FreeBSD kernel to use COM2 (the same port as the bootloader and your mini-USB):

```sh
cat > /boot/loader.conf.local << 'EOF'
boot_multicons="YES"
comconsole_port="0x2f8"
EOF
```

### Fix 2: Serial login (persistent, survives reboots)

OPNsense generates `/etc/ttys` with entries for `ttyU0` and `ttyU1` (uppercase U, for USB serial) when `serialusb` is set in its config. However, the actual FreeBSD device nodes are `ttyu0` and `ttyu1` (lowercase u). This mismatch means `getty` is configured for devices that don't exist — so no login prompt appears.

Additionally, OPNsense's `/etc/pam.d/login` doesn't pass the required parameter to its PAM module, causing authentication to fail even if a prompt does appear.

Both issues need to be fixed and made persistent using OPNsense's own mechanisms.

#### 2a: PAM fix — make it persistent via OPNsense template system

OPNsense regenerates PAM config from templates on every boot. Add a `login.pam` template so OPNsense generates a working `/etc/pam.d/login`:

```sh
# Create the login PAM template
cat > /usr/local/opnsense/service/templates/OPNsense/Auth/login.pam << 'EOF'
# $FreeBSD$
#
# login - auth-owner=root authenticate-with=opnsense
#
auth            sufficient      pam_opnsense.so         authtok_prompt=Password:
auth            required        pam_unix.so             no_warn try_first_pass
auth            optional        pam_group.so            no_warn group=wheel root_only fail_safe ruser
auth            optional        pam_lastlog.so          no_warn

account         required        pam_nologin.so
account         required        pam_opnsense.so
account         required        pam_unix.so
account         required        pam_login_access.so

password        required        pam_opnsense.so
password        required        pam_unix.so             no_warn try_first_pass

session         required        pam_lastlog.so          no_warn
session         optional        pam_motd.so
session         optional        pam_mail.so             no_warn quiet
EOF

# Register it in the template targets
echo "login.pam:/etc/pam.d/login" >> /usr/local/opnsense/service/templates/OPNsense/Auth/+TARGETS

# Apply immediately
configctl template reload OPNsense/Auth
```

#### 2b: ttyU symlink fix — make it persistent via OPNsense syshook

OPNsense has a syshook mechanism that runs scripts at specific points in the boot process. Create a script at `99-serial-console` (the high number ensures it runs after OPNsense's own console configuration at `20-freebsd`):

```sh
cat > /usr/local/etc/rc.syshook.d/start/99-serial-console << 'EOF'
#!/bin/sh

# Fix serial console: create ttyU symlinks and fix PAM login config.
# Must run after 20-freebsd which calls system_login_configure().

# OPNsense writes ttyU (uppercase) in /etc/ttys when serialusb=1,
# but the actual FreeBSD device nodes are ttyu (lowercase).
ln -sf /dev/ttyu0 /dev/ttyU0
ln -sf /dev/ttyu0.init /dev/ttyU0.init
ln -sf /dev/ttyu0.lock /dev/ttyU0.lock
ln -sf /dev/ttyu1 /dev/ttyU1
ln -sf /dev/ttyu1.init /dev/ttyU1.init
ln -sf /dev/ttyu1.lock /dev/ttyU1.lock

# Reload PAM templates to ensure login.pam is applied
configctl template reload OPNsense/Auth

# Signal init to re-read /etc/ttys and start gettys on the serial ports
kill -HUP 1
EOF

chmod +x /usr/local/etc/rc.syshook.d/start/99-serial-console
```

### Verifying it works

Reboot the device. You should see:

1. Full OPNsense boot sequence on your serial terminal (services starting, syshook output, etc.)
2. `>>> Invoking start script 'serial-console'` followed by `OK` — confirming the syshook ran
3. The OPNsense banner and a `login:` prompt on `ttyU1`
4. Successful login with your `root` credentials

From this point the device is fully headless — no GPU, no monitor, serial console only.

### Findings:

* The theoretical throughput of this device, while using all CPUs onboard, is ~750mbps give or take. Single-core performance is ~350-500mbps.

<img src="https://raw.githubusercontent.com/PhoenixSheppy/VeloCloud-Edge-510-OPNsense-Conversion-Guide/refs/heads/main/pics/single_core_test.jpg" height="250">

<img src="https://raw.githubusercontent.com/PhoenixSheppy/VeloCloud-Edge-510-OPNsense-Conversion-Guide/refs/heads/main/pics/multi_core_test.jpg" height="250">

* The M-key Slot on the motherboard does not work, as far as I know it does not show up in any OS I've tested, and cannot be used to boot from. (It does work with a PCie LTE Card on the 'LTE' SKUs!)

* Speaking of LTE, yes, the LTE Sierra Wireless Snapdragon-X7 LTE-A card that comes in the -LTE SKUs does work with OPNSense!

* The wireless-AC also works with OPNsense in the event that you choose to keep it!

* If you're worried about the internal flash dying, if/when it dies, boot priority is pre-configured to boot from USB first, so it'll work fine with a slim-USB drive in a pinch.

* Patches / updates are about as slow as updating a Palo-Alto 220, slow and grueling, but that's mostly a limitation of the onboard eMMC. Flash drives (might) be faster.
