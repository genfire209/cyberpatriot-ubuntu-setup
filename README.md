# CyberPatriot Image Setup on Ubuntu

Last updated Oct 16, 2026

## Before you start

You'll end up with the CyberPatriot image extracted in `~/cp19` and open in VMware Workstation Pro on an Ubuntu PC. Copy each command on its own and press Enter after it; every command here fits on one line.

| You need | Why |
| --- | --- |
| A 64-bit Intel or AMD PC | The image is an x86 VM. Apple M-series Macs and other ARM machines can't run it. |
| Ubuntu 22.04, 24.04 or 26.04 | Other distros work, but the package names below are for Ubuntu. |
| Virtualization turned on in BIOS (Intel VT-x or AMD-V) | VMware won't start a VM without it. |
| 8 GB RAM or more | The Mint 21 practice image uses 4 GB RAM and 2 CPU cores. |
| Free disk space about 4x the zip size | The Mint 21 practice zip is 2.6 GB and extracts to 8.2 GB, and both sit on disk until you delete the zip. |
| Your coach's email | It has the download link, the zip password, the checksum and your team UID. |

Check virtualization is on. A number above 0 means it's on; 0 means turn it on in your BIOS settings.

```
grep -cE 'vmx|svm' /proc/cpuinfo
```

## Step 1: Install the tools

These commands install the build tools VMware needs to compile its kernel modules, plus 7-Zip, aria2 for faster downloads, and mokutil for checking Secure Boot. Open a terminal with Ctrl+Alt+T and run them one at a time. sudo asks for your password, and nothing shows on screen while you type it.

```
sudo apt update
```

```
sudo apt install build-essential linux-headers-$(uname -r) p7zip-full aria2 mokutil
```

On Ubuntu 24.04 and later, `p7zip-full` pulls in the newer `7zip` package automatically. Either way you get the `7z` command.

## Step 2: Install VMware Workstation Pro

VMware Workstation Pro is free, but you need a free Broadcom account to download it.

1. Go to the [Broadcom Workstation Pro downloads page](https://support.broadcom.com/group/ecx/productdownloads?subfamily=VMware%20Workstation%20Pro&freeDownloads=true) and sign in, or sign up first.
2. Open the newest **VMware Workstation Pro for Linux** release.
3. Click the license link and tick the checkbox to accept it. Fill in the address details it asks for (country, city, zip code).
4. Download the file ending in `.bundle`. It saves to your Downloads folder.

Then install it from a terminal:

```
cd ~/Downloads
```

```
chmod +x VMware-Workstation-*.bundle
```

```
sudo ./VMware-Workstation-*.bundle
```

When it says the installation succeeded, open **VMware Workstation** from your app menu. If it asks to compile kernel modules, click Install and enter your password. If it asks how you'll use it, pick personal use.

If VMware doesn't start or complains about modules, build them by hand:

```
sudo vmware-modconfig --console --install-all
```

If that fails with a signing or "/dev/vmmon" error, go to Step 3.

## Step 3: Fix Secure Boot (only if VMs won't start)

If Secure Boot is on, Ubuntu blocks VMware's unsigned kernel modules, and starting a VM fails with "Could not open /dev/vmmon". Check whether it's on:

```
mokutil --sb-state
```

If it says `SecureBoot disabled`, skip this step. If it's enabled, pick one fix.

**Easiest:** turn Secure Boot off in your BIOS/UEFI settings, then run the `vmware-modconfig` command from Step 2 again.

**Keep Secure Boot on:** sign the modules with your own key. This is Broadcom's official procedure ([KB 315309](https://knowledge.broadcom.com/external/article/315309/cannot-open-devvmmon-no-such-file-or-dir.html)).

1. Make a folder for the key and go into it:

   ```
   mkdir -p ~/vmware-key && cd ~/vmware-key
   ```
2. Create the key:

   ```
   openssl req -new -x509 -newkey rsa:2048 -keyout MOK.priv -outform DER -out MOK.der -nodes -days 36500 -subj "/CN=VMware/"
   ```
3. Sign both modules:

   ```
   sudo /usr/src/linux-headers-$(uname -r)/scripts/sign-file sha256 ./MOK.priv ./MOK.der $(modinfo -n vmmon)
   ```

   ```
   sudo /usr/src/linux-headers-$(uname -r)/scripts/sign-file sha256 ./MOK.priv ./MOK.der $(modinfo -n vmnet)
   ```
4. Register the key. It asks you to make up a one-time password; remember it for the next step.

   ```
   sudo mokutil --import MOK.der
   ```
5. Reboot. A blue **MOK Manager** screen appears. Choose Enroll MOK, then Continue, then Yes, type the password from step 4, and reboot.

After an Ubuntu kernel update, VMware rebuilds its modules unsigned. When that happens, `cd ~/vmware-key` and run the two sign commands from step 3 again. You don't need to repeat steps 4 and 5.

## Step 4: Download the image

The simplest way is to click the link in your coach's email and let your browser save the zip to Downloads. The examples below use the Mint 21 practice image, `cp19_pr_e_mint21.zip`; swap in your image's file name.

If the browser download is slow, use aria2 instead. It opens 16 connections at once. On the same download server it got 2-4 MB/s where a single connection got 0.15 MB/s.

```
cd ~/Downloads
```

```
aria2c -x16 -s16 "PASTE-THE-LINK-FROM-THE-EMAIL"
```

Wait for `Download complete` before going on. While aria2 is running, the file can look full-size even though it isn't finished.

## Step 5: Verify the checksum

A matching checksum proves the zip downloaded completely and isn't corrupted. Do this before extracting, in a terminal that's in your Downloads folder (`cd ~/Downloads` from Step 4); Step 6 runs from there too. Paste the checksum from your coach's email in place of the placeholder; there are exactly two spaces before the file name.

```
echo "PASTE-CHECKSUM-HERE  cp19_pr_e_mint21.zip" | md5sum -c
```

`cp19_pr_e_mint21.zip: OK` means it matches. `FAILED` means delete the zip and download it again.

If the email gives a SHA-1 or SHA-256 checksum instead of MD5, use `sha1sum` or `sha256sum` in place of `md5sum`. To just print the value and compare by eye:

```
md5sum cp19_pr_e_mint21.zip
```

## Step 6: Extract with 7-Zip

The zip is password-protected. 7-Zip asks for the password when it needs it, and nothing shows on screen while you type. Copy the password carefully from your coach's email.

First, test the zip. This reads every file without writing anything and ends with `Everything is Ok`. `Wrong password` means retype it.

```
7z t cp19_pr_e_mint21.zip
```

Then extract it into `~/cp19`:

```
mkdir -p ~/cp19
```

```
7z x cp19_pr_e_mint21.zip -o"$HOME/cp19"
```

The Mint 21 practice image takes a minute or two and ends with `Everything is Ok`, 10 files, about 8.2 GB. Other images have different sizes. Check the result:

```
ls ~/cp19/*
```

You should see a `.vmx` file and several `.vmdk` disk files. Once that works, you can delete the zip to free space, but you'll have to download it again if you ever need to re-extract:

```
rm ~/Downloads/cp19_pr_e_mint21.zip
```

## Step 7: Open it in VMware

Open the `.vmx` file from VMware (File, then Open, then go to `~/cp19`) or from a terminal:

```
vmware ~/cp19/cp19_pr_e_mint21/cp19_pr_e_mint21.vmx
```

If VMware asks whether you moved or copied the VM, pick **I Copied It**. That's what the CyberTaipan guide says, and CyberTaipan uses the same images. Then click Power On.

On first boot the image asks for your team's **UID** from your coach. The PC needs internet so the scoring report can update.

Rules for the image:

- Keep the VM's network on **NAT**.
- Don't move, rename or edit any files in the image folder.
- Only one copy of each image can be open at a time.
- Images can expire a few days after extraction; the Training Round image expired after 7 days. If yours expires, download and extract it again.

## Troubleshooting

| Problem | Fix |
| --- | --- |
| "Could not open /dev/vmmon" | Secure Boot is blocking the modules; do Step 3. If Secure Boot is off, rebuild with `sudo vmware-modconfig --console --install-all`. |
| VMs stopped starting after an Ubuntu update | The kernel changed. Run `sudo apt install linux-headers-$(uname -r)`, then the `vmware-modconfig` command. With Secure Boot on, re-sign (Step 3, step 3). |
| Modules won't compile on a brand-new kernel | Install the newest VMware release from Broadcom. If there's no fix yet, use the patched modules from [philipl/vmware-host-modules](https://github.com/philipl/vmware-host-modules). |
| "This host does not support Intel VT-x" or AMD-V | Turn on virtualization in your BIOS settings. |
| 7-Zip says `Wrong password` | Re-copy the password from the email; watch for look-alike characters such as O and 0, or l and 1. |
| 7-Zip reports CRC or data errors | The download is damaged. Delete the zip, download again, and check the checksum. |
| Checksum says `FAILED` | Delete the zip and download again. |
| Not enough disk space | Check with `df -h ~`. You need about 4x the zip size free. |
| Browser download is very slow | Use aria2 (Step 4). |
| Scoring report doesn't update | Make sure the VM's network is NAT and the PC is online. |

## After the round: delete the image

Your coach requires images to be removed when the round ends.

1. Shut down the VM and close VMware.
2. In VMware's Library sidebar, right-click the VM and choose Remove so it drops off the list.
3. Delete the image folder and any zip left in Downloads:

   ```
   rm -rf ~/cp19/cp19_pr_e_mint21
   ```

   ```
   rm -f ~/Downloads/cp19_pr_e_mint21.zip
   ```

Keep VMware installed for the next round. To remove it entirely: `sudo vmware-installer -u vmware-workstation`.

## Sources

- [How to Install VMware Workstation Pro 26H1 in Ubuntu 26.04, 24.04](https://ubuntuhandbook.org/index.php/2024/04/install-vmware-player-ubuntu/), UbuntuHandbook, last updated May 18, 2026
- [KB 315309: "Cannot open /dev/vmmon" error when powering on a VM](https://knowledge.broadcom.com/external/article/315309/cannot-open-devvmmon-no-such-file-or-dir.html), Broadcom
- [VMware Workstation Pro downloads](https://support.broadcom.com/group/ecx/productdownloads?subfamily=VMware%20Workstation%20Pro&freeDownloads=true), Broadcom Support Portal
