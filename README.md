# Fixing black patches and lag: Mac Windows App → Linux xrdp

Field notes from getting a smooth RDP desktop out of this setup:

- **Client:** Microsoft Windows App (formerly Microsoft Remote Desktop) on macOS
- **Server:** Ubuntu 24.04, xrdp 0.9.24, XFCE

Most of it applies to other xrdp setups too. If your RDP host is **Windows**, skip to [the last section](#if-the-host-is-windows-not-linux).

## TL;DR

1. Turn off the desktop's compositor.
2. Set `max_bpp=24` in `/etc/xrdp/xrdp.ini`, **while nobody is logged in**.
3. Optional, for speed: raise xrdp's TCP send buffer.
4. Never restart xrdp or change color depth while someone has a desktop open. See [the two traps](#traps-password-prompt-then-nothing).

## Symptoms

- Large black areas, parts of the desktop never repainted, dialogs that look washed out or see-through.
- Small black rectangles that show up now and then while working.
- Sluggish redraws, especially when moving windows or scrolling.

## Why it happens

- **No codec in common.** The Mac client advertises only NSCodec, which xrdp 0.9 can't encode. `/var/log/xrdp.log` shows `xrdp_caps_process_codecs: nscodec`. So xrdp falls back to sending a compressed bitmap of every screen change, and there's no video-style encoding.
- **Compositing window managers** repaint large areas constantly, for shadows and transparency, and are known to leave unpainted regions under xrdp.
- **32-bit color** bitmaps carry an alpha (transparency) channel. The likely explanation for the intermittent black patches is regions arriving marked fully transparent. Limiting to 24-bit makes them go away. This is the commonly reported workaround; I didn't verify the exact mechanism.

## Fix 1: turn off the compositor

This fixed the large black areas and the see-through windows. It also makes everything snappier.

**XFCE, inside the session:** Settings → Window Manager Tweaks → Compositor → uncheck **Enable display compositing**. Or run:

```sh
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

**XFCE, over SSH, for a desktop that's already running** (it needs that session's D-Bus address):

```sh
PID=$(pgrep -u "$USER" -x xfce4-session | head -1)
export $(tr '\0' '\n' < /proc/$PID/environ | grep -E '^(DBUS_SESSION_BUS_ADDRESS|DISPLAY)=' | xargs)
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

The change takes effect immediately and persists across sessions.

**Other desktops** (not tested here): on KDE Plasma, go to System Settings → Display and Monitor → Compositor. On MATE, run `gsettings set org.mate.Marco.general compositing-manager false`.

## Fix 2: cap color depth at 24-bit

This fixed the remaining intermittent black patches.

In `/etc/xrdp/xrdp.ini`, under `[Globals]`:

```ini
max_bpp=24
```

xrdp reads this on every new connection, so **no restart is needed**. But the user **must be logged out of the desktop first**, not just disconnected. Otherwise you hit [Trap 1](#trap-1-session-matching-includes-color-depth).

Safe procedure:

1. The user logs out (in XFCE, Applications → Log Out). Closing the RDP window isn't enough.
2. Edit `xrdp.ini`.
3. The user reconnects. `/var/log/xrdp.log` should show:
   `Client requested 32 bpp color depth, but the server configuration is limited to 24 bpp.`

To have the switch happen automatically the moment the user logs out, so you don't have to coordinate:

```sh
sudo systemd-run --unit=xrdp-bpp-switch --property=RuntimeMaxSec=3600 /bin/bash -c \
  'while pgrep -u USERNAME -x xfce4-session >/dev/null; do sleep 1; done
   sed -i -E "s/^max_bpp=32/max_bpp=24/" /etc/xrdp/xrdp.ini'
```

On the client, leave color quality at its highest setting. Picking a lower one creates its own color-depth mismatch.

## Fix 3 (speed): bigger TCP send buffer

The default socket buffer, about 200 KB, limits throughput once there's real latency between you and the server. This doesn't fix artifacts; it makes redraws faster.

In `/etc/xrdp/xrdp.ini`, under `[Globals]`:

```ini
tcp_send_buffer_bytes=4194304
```

Let the kernel allow that size:

```sh
echo 'net.core.wmem_max = 8388608' | sudo tee /etc/sysctl.d/60-xrdp.conf
sudo sysctl -p /etc/sysctl.d/60-xrdp.conf
```

xrdp reads this setting only at startup, so it needs `sudo systemctl restart xrdp`. **Restart only when nobody is logged in.** See [Trap 2](#trap-2-restarting-xrdp-orphans-running-desktops).

## Client-side tips

- **Don't resize the RDP window mid-session.** Each resize makes xrdp renegotiate the screen size and can leave black areas until the next full redraw. `~/.xorgxrdp.N.log` shows `screen resized to WxH` each time.
- **Use a fixed resolution**, such as 1920×1080, instead of a huge native one. Fewer pixels means less data per update.

## Traps: password prompt, then nothing

Both traps look the same from the Mac. You enter your password, then the client sits there forever with **no error message**.

### Trap 1: session matching includes color depth

xrdp's session manager (sesman) reattaches you to your existing desktop only if **both the user and the color depth match**. In 0.9, `Policy=Default` means user + bpp. If the depth differs, it starts a *second* desktop for the same user. XFCE won't run twice for one user, so the new desktop exits within a second, and the client hangs.

`/var/log/xrdp-sesman.log`:

```
Starting session: ... display :11.0, ... bpp 24 ...
Window manager (pid ..., display 11) exited quickly (1 secs). This could indicate a window manager config problem
```

The giveaway is the display number going up (`:11` instead of the existing `:10`).

### Trap 2: restarting xrdp orphans running desktops

On Ubuntu, `xrdp-sesman.service` has `BindsTo=xrdp.service` and `StopWhenUnneeded=true`. So `systemctl restart xrdp` restarts sesman too, and sesman forgets every running desktop. The desktops keep running but can never be reattached, so every new login runs into Trap 1.

Check:

```sh
systemctl cat xrdp-sesman | grep -E 'BindsTo|StopWhenUnneeded'
```

### Recovering from either trap

End the stuck desktop. This closes any apps still running in it.

```sh
loginctl list-sessions                    # xrdp desktops show up as "cN" sessions with no TTY
sudo pkill -u USERNAME -x xfce4-session
sudo loginctl terminate-session c1        # the stale cN session
# if something lingers in that session:
sudo kill $(cat /sys/fs/cgroup/user.slice/user-$(id -u USERNAME).slice/session-c1.scope/cgroup.procs)
```

Then reconnect. You'll get a fresh desktop.

## Diagnostics cheat sheet

| Log | Line | Meaning |
| --- | --- | --- |
| `/var/log/xrdp.log` | `xrdp_caps_process_codecs: nscodec` | Client offered no codec xrdp 0.9 can use, so it sends plain bitmaps |
| `/var/log/xrdp.log` | `Client requested 32 bpp ... limited to 24 bpp` | Fix 2 is active |
| `/var/log/xrdp.log` | `SSL: ... unexpected eof while reading` | Client disconnected or the window was closed; harmless |
| `/var/log/xrdp-sesman.log` | `Window manager ... exited quickly (1 secs)` | Duplicate desktop; see Trap 1 or 2 |
| `/var/log/xrdp-sesman.log` | `sesman_data_in: scp_process_msg failed` | Appears on every login in 0.9.24; harmless |
| `~/.xorgxrdp.N.log` | `screen resized to WxH` | The client resized the window |

## If it's still bad: xrdp 0.10

xrdp 0.10.x adds the GFX pipeline, with H.264 and RFX progressive, which the Mac client prefers. It's a different and much more efficient way of sending the screen, especially for scrolling and video. Ubuntu 24.04 ships 0.9.24, so getting 0.10 means building xrdp and xorgxrdp from source or using a newer distro. I haven't tested this here.

## Baseline setup (Ubuntu 24.04)

```sh
sudo apt install xfce4 xfce4-goodies xrdp dbus-x11
sudo adduser xrdp ssl-cert                    # lets xrdp read the TLS key
echo xfce4-session > ~/.xsession              # for each RDP user
sudo systemctl disable --now lightdm          # no local display manager needed on a server
sudo systemctl set-default multi-user.target
```

## If the host is Windows, not Linux

Most of the above doesn't apply. A common culprit for black or half-drawn screens with Windows hosts is RDP's UDP transport going over a VPN that can't carry full-size packets. Test with TCP only:

- **Host:** Group Policy → Computer Configuration → Administrative Templates → Windows Components → Remote Desktop Services → Remote Desktop Session Host → Connections → **Select RDP transport protocols** → *Use only TCP*.
- **Windows clients:** the **Turn Off UDP On Client** policy.

This is general knowledge, not something verified in this setup.
