# NixOS on CM3855 with ZFS – Quick Install Guide

I initially wrote this as a personal reminder in case I needed to reinstall my board, but since it was quite a journey to get everything working, I hope it can help others as well.

> **Note:** This is not necessarily the only or best way to install NixOS on this hardware — it's simply what worked for me.  
> Much of this information was gathered from multiple scattered sources, which I’ve compiled here into a complete guide. I listed some of them at the end of this Readme.  

## 🧰 Create a NixOS image

We first need to build a compatible image for the **CM3855** board.  
We’ll use the [`nixos-aarch64-images`](https://github.com/Mic92/nixos-aarch64-images) repository, which contains preconfigured builds for ARM boards like the **CM3588**.

<br>

- Install Nix, follow the installation instructions here: [https://nixos.org/download/](https://nixos.org/download/)
- Build the image
 ```bash
  nix build --no-write-lock-file 'github:Mic92/nixos-aarch64-images#cm3588NAS' \
     --extra-experimental-features nix-command \
     --extra-experimental-features flakes
 ```
After the build finishes, you’ll get a file named `result`, this is your NixOS image.

- Flash the result file to an external drive (USB stick, SD card...) For this tutorial, we’ll assume you’re using an SD card.
> I personally used **Etcher**, but had to add a random extension to the `result` file (e.g., .img) for Etcher to recognize it.

<br>

Now you should have a bootable SD card with NixOS Live.
However, since HDMI output won’t work, you’ll need to access the board remotely via SSH.

## 🔑 Enable SSH Access on the Live Image

- Plug the SD card back into your computer (mount it if it doesn’t mount automatically).
- Navigate to the SSH configuration directory:  `cd etc/ssh/`
- Create a new folder : `mkdir authorized_keys.d`
- Copy and paste your public SSH key in a file called `root` :  `nano authorized_keys.d/root`


## 💾 Copy the SD Card Content to the eMMC
- Insert the SD card into the CM3588.
- Boot the board, then connect remotely via SSH: `ssh root@<the_cm3588_ip>`
- Run `lsblk` to identity the eMMC and the SD card. For this guide, we’ll assume that the eMMC is `/dev/mmcblk0` and the SD card is `/dev/mmcblk1`
- Copy the SD card content to the eMMC (with all partitions) : `sudo dd if=/dev/mmcblk1 of=mmcblk0 bs=4M status=progress`

Once the copy is done:

- Shut down the CM3588
- Remove the SD card
- Power it back on — it should now boot into the live NixOS environment from the eMMC

## ⚙️ Install NixOS on eMMC
- Resize the main partition to use all available space on the eMMC:
```
parted /dev/mmcblk1 rm 4
parted /dev/mmcblk1 resizepart 3 100%
resize2fs /dev/mmcblk1p3
```
- Then mount the main partition : `mount /dev/mmcblk1p3 /mnt`
- Generate the NixOS configuration : `nixos-generate-config --root /mnt`
- Edit the `/mnt/etc/nixos/configuration.nix` file and make sure it contains SSH configuration. For example with a SSH key :
```
{
  # ...
  networking.hostName = "cm3588";

  services.openssh = {
    enable = true;
    settings.PasswordAuthentication = false;
  };

  users.users.root.openssh.authorizedKeys.keys = [
    "<your_public_key>"
  ];
}
```
- Finally, install NixOS : `nixos-install --root /mnt`
- Reboot and everything should be good to go !

## 🔗 References
- https://wiki.nixos.org/wiki/NixOS_on_ARM/FriendlyELEC_CM3588#Installation
- https://blog.psychollama.io/nixos-on-a-cm3588/
