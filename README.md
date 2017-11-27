# PRIME support scripts

On my W530 with Debian, Bumblebee didn't want to play with the nouveau driver, I did not
want to install the proprietary nVidia drivers, the external connectors are powered by
the discrete GPU and I did not want to have that enabled all the time for battery conservation.

This collection of scripts and configuration allows me to start the GPU when I need it,
including the external connectors, but still have it disabled on boot saving battery.

Applications can be run on the discrete GPU using [PRIME](https://wiki.archlinux.org/index.php/PRIME)
functionality.

Though it is pretty hacky, it should work alright with minor hickups.

## What does what

I'll provide a short explanation of the purposes of the respective files

### `/etc/modules-load.d/prime.conf`

* Force the `bbswitch` and `i915` modules to be loaded at boot to provide power
	switching for the discrete GPU as well as a driver for the internal one.

### `/etc/modprobe.d/gpuoff.conf`

* Prevents `nouveau` from getting loaded (limiting us to the intel card), so the discrete
	GPU does not power on
* Adds options to the `bbswitch` module to turn the card off on boot

### `/usr/local/sbin/load-nouveau`

* Loads the `nouveau` module (blacklisted at boot), starting the discrete GPU

### `/etc/sudoers.d/gpu`

* Allows users of the `video` group to run the `/usr/local/sbin/load-nouveau` as root.
	Making the script `setuid` would work, too, but this lets us limit execution to
	group members.

## Setup

* Copy the configuration files to the appropriate locations
* Make sure `bbswitch` is installed and the module can be loaded
	* Test this by running `modprobe bbswitch` and/or `modinfo bbswitch`
	* Under Debian, installing `bbswitch-dkms` only provides you with the source, to actually
		build and use the module you need
		* the kernel-headers package for your current kernel installed
		* to run `dpkg-reconfigure bbswitch-dkms` to build and install the module
* Install the `gpustart` function and optionally the `gpu` alias (see below)
* Reboot your machine
* Check if the discrete card is off by looking at `/proc/acpi/bbswitch`
* Use the scripts as described below

### Add start function to shell initialization

In my case, I add the following lines to `~/.zshrc`:

```
alias gpu="DRI_PRIME=1"
gpustart(){
        sudo load-nouveau
        xrandr --setprovideroutputsource 1 0
        xrandr --setprovideroffloadsink 1 0
}
```

If `xrandr --listproviders` outputs useful names for you, you may replace the last two parameters
to the `xrandr` calls with proper names. Otherwise you may need to figure out the correct order of
the two cards with [this guide](https://wiki.archlinux.org/index.php/PRIME).

## Usage

* From within a running X server, run the `gpustart` function. On some `intel` drivers, this crashes
	the running X server, though with the KMS intel driver (which has other problems), this seemed
	to work seamlessly.
* If your X server dies, just start a new one and run the function again. You should now have access
	to the discrete card's outputs.
* To run applications on the discrete card, set the `DRI_PRIME` environment variable to `1` before running
	the target application. Alternatively, use the `gpu` alias.

## Helpful

The current discrete card power status can be seen in `/proc/acpi/bbswitch`.
Writing `OFF` to that file turns off the power to the card.

To see which card is being used to render, run `glxinfo | grep -i opengl` or `glxgears -info`.

## Problems

Switching the card off again after usage is currently not implemented, but would involve
stopping all PRIME clients, removing the sink assignments, switching off the card and unloading
the `nouveau` driver kernel module in approximately that order.

The module load order in `/etc/modules-load.d/prime.conf` is important, as loading `nouveau` before
the driver providing your display (eg `i915`), or indeed not loading it at all, will leave you with
outputs named `LVDS-2`.

## Ideas / Feedback / Future work

Some sources say that some (newer?) versions of nouveau provide internal power control,
switching the card off when it is no longer required. This would allow me to stop using
bbswitch for that purpose.
