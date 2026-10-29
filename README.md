# Spotify's picture helper crashes on Omarchy

Spotify stays open, then the part that draws the window dies. The song can keep going. Album art, the canvas, or a music video drops out. It comes back, then dies again.

This is a note from one Omarchy machine in October 2026. The setting below stopped the crash there. It is not an official Spotify or Omarchy fix.

## Fix

Quit Spotify completely. A window you can still see means the old copy is still running, and the new setting will not apply until that copy is gone.

Create this file:

`~/.config/spotify-flags.conf`

Put this line in it, with nothing else:

```
--disable-gpu
```

Open Spotify from the app menu.

The Arch Spotify launcher reads that file on every start and passes the line through. Spotify then draws the window in software instead of using the graphics chip. The drawing helper still starts, but it is told not to use the chip (`--use-gl=disabled`). On the machine this was tried on, that copy stayed up and playback looked normal.

The cost is that pictures and video use more of the processor. If you would rather have the old behavior, delete `~/.config/spotify-flags.conf` and open Spotify again.

## Why it crashes

Omarchy runs a Wayland session and asks apps to use Wayland. Spotify 1.3.4.258 does not. Its drawing helper starts in X11 mode (`--ozone-platform=x11`) and hands the graphics library a display object from that mode.

The graphics library (Mesa) then takes the Wayland path anyway. Wayland tries to lock that object as if it were a Wayland display. It is not one. The field that should point at a Wayland display was the number 11, so the lock touched memory that is not there. The helper is killed for that bad read (SIGSEGV, SEGV_MAPERR).

The call that dies is Wayland's `wl_proxy_create_wrapper` (`wayland-client.c` line 2801), locking `display->mutex`. Mesa's EGL library makes that call. Under it is Spotify's private copy of Chrome (`libcef.so`, Chrome 151). That copy has no public function names.

The same helper died four times on 22 October 2026 and again on 29 October, with the same stack each time. It was not out of memory. Updating the machine the day before did not change Spotify, Mesa, or Wayland.

Versions on that machine:

- Spotify 1.3.4.258
- Mesa 26.2.5
- Wayland 1.26.0
- Intel graphics

Your songs and playlists are in your Spotify account. This crash does not delete them.

## What did not work

A personal launcher that set `EGL_PLATFORM=x11` did reach the main Spotify process. It did not reach the drawing helper. Spotify clears the environment before that helper starts. The helper crashed again within half a minute, on the same Wayland lock.

Forcing Spotify onto Wayland with `--ozone-platform=wayland` is a different workaround other people have used for scaling. It was not the one tested here. Omarchy's maintainer has also found Spotify's Wayland layout too wide to be worth it for everyday use.

## This is Spotify's bug

Omarchy installs Spotify and sets a Wayland preference. The crash is in the Chrome copy Spotify ships, when Mesa is given a display that is not a Wayland display. Changing Omarchy's Wayland preference would affect every app, and Spotify's helper already forces X11.

The same crash, on Spotify 1.3.4.258 under Wayland, is reported on Spotify's own forum:

https://community.spotify.com/forum/en/discussion/7561263/linux-gpu-process-sigsegv-after-updating-to-spotify-1-3-4-258-x11-wayland-display-mismatch

That is the place to tell Spotify. A report to Omarchy would send the maintainers a bug in an app they do not write.

## How to see if you have this crash

```bash
coredumpctl list | grep spotify
```

Look for SIGSEGV on `/opt/spotify/spotify`. The details should show `--type=gpu-process` and `--ozone-platform=x11`, and the top of the stack in `wl_proxy_create_wrapper`.
