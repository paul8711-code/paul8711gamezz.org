Essential before all of this:

Load the correct keymap for you, for example `loadkeys de-latin1` to load a german keyboard layout.

#### 1. Connect to internet
You can do this either via LAN or iwctl

If you choose iwctl:

First, enter the interactive prompt:
```bash
iwctl
```
Next, if you do not know your wireless device name, list all devices:
```bash
device list
```
After that, scan for networks (will not output anything):
```bash
station <name> scan
```
Now you can list available networks:
```bash
station <name> get-networks
```
And then just connect to the network:
```bash
station <name> connect <SSID>
```
Or if your network is hidden:
```bash
station <name> connect-hidden <SSID>
```

#### 2. Partition your drive

In this step, we will use `cfdisk` to create the required partitions.

After executing the command, you might get prompted to choose between MBR (often shown as "dos") and GPT (and maybe more).

If you choose GPT (which I recommend), the partitioning will be a bit easier. The process should be the same, though.

If you did not get prompted, that is perfectly fine. I will explain both anyways.

Use the arrow keys to navigate to the option "New", or, if you do not have "Free space", first resize/delete partitions (and then select "New").

This first partition will be your boot partition, it normally does not have to be more than 512 MB.

Now select "New" again. This partition will be your swap partition, if you do not plan on having that, just skip this step (the rest will be basically the same).

Here I recommend allocating as much storage space to swap as you have RAM.

Create a new partition one last time. This will be your main data partition, you can allocate as much as you want here (as long as it is not too small).

Also make sure to write down or remember which partition is which (for example /dev/sda1 for boot)

After doing all that, just select "Write" and once that is done, "Quit".

If this failed or gave you a selection between a "primary" and "extended" partition, your system uses MBR. While you could reformat your ENTIRE disk to use GPT, there is an easier solution.

MBR is (in my eyes) worse, because it only supports a maximum of 4 primary partitions. You can avoid this, by creating an extended partition. In that extended partition, you can create basically infinite "normal" partitions.

Now, we need to format (and mount) the partititions, so Arch likes them.

Let's start with the partition you made last:
It needs to be formatted as ext4. You can achieve this by running the below command.
```bash
mkfs.ext4 /dev/sda3
```
Make sure to replace sda3 with your actual partition.

After that, format the boot partition as fat32 like this:
```bash
mkfs.fat -F 32 /dev/sda1
```
Once again, replace sda1 with your actual partition.

Last we format the swap partition as swap:
```bash
mkswap /dev/sda2
```

If you now run the command `lsblk` you should see that your partitions are indeed formatted like I said.

Now we mount the partitions:
```bash
mount /dev/sda3 /mnt
mkdir -p /mnt/boot/efi
mount /dev/sda1 /mnt/boot/efi
swapon /dev/sda2
```

And that was already the last step of this chapter. Now we move on to

#### 3. Installing the base system

Now that all partitions are mounted and formatted, we can use `pacstrap` to download some essential system... things.

After that, we will also configure some stuff, like the time zone.

This command downloads and installs at least a basic set of tools for us to use/configure later:
```bash
pacstrap -K /mnt base linux linux-firmware sof-firmware alsa-utils base-devel grub efibootmgr nano networkmanager
```
Mandatory packages are: `base`, `linux` and `linux-firmware`.

`sof-firmware` is used (specifically for Intel-based devices) to make your speakers and microphone work.

`alsa-utils` is also audio related.

`base-devel` includes tools like gcc, make, sudo and many more.

`nano` is a simple command line text editor.

`networkmanager` makes it easier to connect to the internet (so we dont have to do the witchcraft from the beginning again).

`grub` and `efibootmgr` are both for booting.

You can also leave some non-mandatory out, but my guide assumes you installed all of them.


After the command finished downloading and installing, we can start with fstab.

This will basically tell your system, which partition it should boot from and which to treat as root.

```bash
genfstab -U /mnt > /mnt/etc/fstab
```
The part before the arrow would output something to the console, the arrow "redirects" the output into the path we specified (gets saved to file).

Now, we can "enter" our system for the first time:
```bash
arch-chroot /mnt
```
This is now your "real" system and not the install device like before.

Here our first step is: Setting the timezone.
```
ln -sf /usr/share/zoneinfo/<Region>/<City> /etc/localtime
```
Make sure to replace region and city with your actual region and city.

Now you can run the `date` command and check if the output is correct.

After that run the following command to synchronize the hardware clock to your system clock:
```bash
hwclock --systohc
```

Next, we'll generate the locale, so the display language of your system.

Use `nano` or any other text editor of choice for this:
```bash
nano /etc/locale.gen
```
In the file, use the arrow keys to scroll down to the locale you want and remove the "#".

For example for US English: Remove the "#" at this line: en_US.UTF-8 UTF-8

Then close `nano` again (CTRL+O then CTRL+X).

Now run `locale-gen` to generate your locale configuration.

Once that is done, we need to specify the language we want to use (since you could generate multiple locales):
```bash
nano /etc/locale.conf
```

Continuing my example from before, I would write into that file:
`LANG=en_US.UTF-8`

Now if you want to set a different keymap as default:
`nano /etc/vconsole.conf` and write `KEYMAP=de-latin1` (for example).

Before going to the next step, we still need to do two things:
- Setting our hostname
- and setting our root password

For the first one, we use `nano /etc/hostname`. Here you can enter whatever you want.

The root password is also fairly straight forward: Just run `passwd` and enter the password you want.

And now, we can advance to

#### 4. Setting up users

In this step I will be setting up a user "paul8711" (because it is my guide). You can replace the username with anything you like, though.

So first we create the user:
```bash
useradd -m -G wheel -s /bin/bash paul8711
```
Next we set a password for the user, like so: `passwd paul8711` and enter the password we want again.

Now you as a user probably want access to `sudo`, so we run `EDITOR=nano visudo`.

There we scroll down to where it says `%wheel ALL=(ALL) ALL` (or ALL:ALL or similar) and uncomment that line.

Do NOT uncomment any line that says something with NOPASSWD unless you want to skip passwords when using `sudo` (security risk).

#### 5. Finishing the Installation

This is the last step before you are done!

First, we enable Networkmanager: `systemctl enable NetworkManager`

Then we install grub to our disk: `grub-install /dev/sda`

And generate a config:
```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

Then just `exit`, `umount -R /mnt` and `reboot`. At this point, you can remove your boot medium.

First, test your internet connection by running `ping paul8711gamezz.org`.

If the connection fails, use `nmtui` or `nmcli` to connect to internet again.

If you do not want to have a desktop environment, skip the following.

We will be installing KDE Plasma as our desktop environment (because it is simple). You can also install a different one.

So first run
```bash
sudo pacman -S plasma sddm
```

After that is done, I recommend you also install Firefox or a different browser: `sudo pacman -S firefox`

And also some kind of terminal: `sudo pacman -S kate`

Last step: Enable sddm and login: `sudo systemctl enable --now sddm`

Congratulations! You have just installed Arch Linux!