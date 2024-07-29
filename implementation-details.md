## Implementation details

If you're not interested in the details, you can skip ahead to the [installation instructions](README.md#installation), but
it might be worhth giving this section a skim so you can see what's going on and fix things if they break later.

This section refers to the modifications performed by the automatic configuration script in this repo. For details
on unlocking the write protect setting and installing linux in the first place, see the [installation instructions](README.md#installation).

### Kernel

The kernel now directly using Chrultrabook pre-compiled kernel from this [repo](https://github.com/chrultrabook/debian-kernel) as documented in Chrultrabook docs page [Install Linux](https://docs.chrultrabook.com/docs/installing/installing-linux.html).

#### Swap support

The chromium fork of the kernel does not support swapping to disk 
[by design](https://www.chromium.org/chromium-os/chromiumos-design-docs/chromium-os-kernel#Swap_6152778388932347_77212105),
to prevent wearing out the flash chips that are commonly used for storage on ChromeOS devices.

It is possible to enable swapping to a portion of ram that's been setup to compress its contents using a kernel feature called 
[zram](https://en.wikipedia.org/wiki/Zram).

The automatic install will add a `/usr/local/sbin/setup-zram-swap.sh` script and a `zram-swap` systemd service
that runs the setup script at boot. By default the script will enable zram-based swap of about 1.5x the size
of physical RAM. I have no idea if this is a good value or not; the script is a minimally altered version of
[the chromiumos swap setup script](https://chromium.googlesource.com/chromiumos/platform/init/+/factory-3536.B/swap.conf).

The size of the zram swap allocation can be controlled by writing an integer value to `/root/.zram-swap`.
Check [the setup script source](ansible/roles/eve-tweaks/files/setup-zram-swap.sh) for valid values. 

#### Hibernation

Hibernation is a power state where the contents of ram are written out to disk, to prevent loss of data in the case of complete
power loss. Because this feature relies on a disk-based swap partition or file, it's not possible to enable while running the
chromium-flavored kernel.

In practice, the suspend state should last for several days before draining the battery completely, which is good enough for me.

### Firmware

Again, this is inspired by `@megabytefisher`. To enable the audio hardware, we need some firmware files that can be
extracted from a Pixelbook recovery image. The automated configuration script will pull all the firmware files from
the recovery image and install them to the correct place, and also pull some configuration needed for full audio
support using `cras`.

### Audio support

Replaced with support using custom kernel that supports the audio setup of Chromebook as implemented in this [repo](https://github.com/WeirdTreeThing/chromebook-linux-audio). The whole solution would not work with Ubuntu after 21.04 as I have tried once before. (Why? I do not know nor having the time to understand why.)

#### Background info

At the bottom is the hardware driver, which takes the form of a kernel module, which in our case also requires some
special firmware files. Once those are in place, the audio hardware is avalable outside of the kernel. However,
the "interface" for using the hardware is extremely low-level and not directly usable by most linux programs and
end users which is now fixed with wireplumber to increase alsa volume headroom.

### Keyboard backlight

When running the chromium-flavored kernel, the keyboard backlight can be controlled by writing an integer value between 0 and 100
to `/sys/class/leds/chromeos::kbd_backlight/brightness`, e.g. `echo 100 > '/sys/class/leds/chromeos::kbd_backlight/brightness'`.
By default only `root` has permission to do this, so the automatic install script installs a udev rules file to grant permission
to an `leds` group, and also makes sure the group exists and the main user account is a member.

There's also a `eve-keyboard-brightness.sh` script installd in `/usr/local/bin` that you can use to more easily set the brightness:

- Set brightness to absolute level between 0 and 100: `eve-keyboard-brightness.sh 100`.
- Adjust brightness by relative amount: `eve-keyboard-brightness.sh +10`
  - handy for assigning to a keyboard shortcut
  
### Touchpad

By default, the pixelbook touchpad feels off in non-ChromeOS linux, requiring too much pressure to move the cursor. This can be fixed by 
[fiddling with libinput debug tools](https://wayland.freedesktop.org/libinput/doc/latest/touchpad-pressure-debugging.html#touchpad-pressure-hwdb)
and creating a `/etc/libinput/local-overrides.quirks` file. 

An earlier version of this repo included a quirks file to set the sensitivity, but I realized I could do one better by
once again building some ChromeOS platform code for vanilla linux.

ChromeOS has its own X11 input driver called `cmt` (for Chromium Multi Touch), which is why the touchpad feels so
much nicer on ChromeOS than most flavors of linux.

I found a [fork of the `xf86-input-cmt` driver by @hugegreenbug](https://github.com/hugegreenbug/xf86-input-cmt/),
which convinced me to try compiling it. Because @hugegreenbug's repo hasn't been updated in a few years, I decided
to make my own patches based on his changes and apply them during the automatic configuration process.

Running the automatic config script will build the input driver and its dependencies and install everything into
the right place. On reboot, you should have nice touchpad sensitivity, and you'll be able to scroll much more
smoothly.

### Remapping non-standard keyboard keys

The Pixelbook keyboard has three non-standard keys, the search key that takes the place of Caps Lock, the "hamburger" key in the upper right, and the Google Assistant key betweeen left control and left alt.
The hamburger key is automatically recognized as F13, and search gets mapped to Left Super, but the assistant key is ignored.

The solution now implement a proper keyboard mapping for almost all Chromebook models based on the file `60-keyboard.hwdb` provided by this [repo](https://github.com/mateowoetam/tuxelbookgoscript) plus a keyboard remapping to fix the offset for the f-keys using keyd configurations.