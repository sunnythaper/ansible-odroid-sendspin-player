# ansible-odroid-sendspin-player

Turns an Odroid C2 running Armbian into a self-managing [Sendspin](https://github.com/Sendspin/sendspin-python-cli) TUI appliance: the sendspin terminal player runs fullscreen on the HDMI display (Now Playing + visualizer), audio plays out of a Hardkernel HiFi Shield 2 (RCA or optical), and each device converges itself via `ansible-pull` on a timer. No control node needed.

```
┌─ Odroid C2 (Armbian minimal, Debian trixie) ───────────────────────┐
│ sendspin-pull.timer ──> ansible-pull ──> local.yml (this repo)     │
│ sendspin-tui.service ──> TUI on tty1 (restart on crash, no blank)  │
│ audio: HiFi Shield 2 (PCM5242 @ 0x4c on I2C-A, I2S on the header) │
└─────────────────────────────────────────────────────────────────────┘
```

## Requirements

- Odroid C2 with a HiFi Shield 2 mounted (RCA to your amp, or optical)
- Armbian minimal image, Debian 13 *trixie*, aarch64
- HDMI display connected (this is the TUI screen)
- Network with internet access on the device

## Bootstrapping a device (per Odroid)

SSH in as your regular user, then:

```bash
sudo hostnamectl set-hostname livingroom          # per-device value
sudo apt-get update && sudo apt-get install -y ansible git
sudo ansible-pull -i livingroom, -U https://github.com/sunnythaper/ansible-odroid-sendspin-player.git -d /var/lib/ansible/local
```

What happens on that first run:

1. The playbook renames the device, installs packages, creates the `sendspin` user, installs `uv`, and installs sendspin with `uv tool install`.
2. The HiFi Shield 2 device-tree overlay is installed to `/boot/overlay-user/hifishield2.dts`, compiled with `dtc`, and activated via `user_overlays=hifishield2` in `/boot/armbianEnv.txt`.
3. The device **reboots automatically** (see `sendspin_allow_reboot`).
4. Roughly 10 minutes after boot the self-installed timer runs the playbook again; the shield's ALSA card is detected, the mixer defaults are applied, `settings-tui.json` is written, and the TUI starts fullscreen on the display.

From then on the device re-converges every 30 minutes and shortly after each boot.

## Per-device configuration

Appliance-wide knobs live in `group_vars/all.yml` — override any of them there (all devices) or in `host_vars/<hostname>.yml` (one device, named after its hostname). Two board-local toggles live in `roles/hifi_shield2/defaults/main.yml`:

```yaml
---
# host_vars/livingroom.yml
sendspin_name: Living Room          # friendly name shown on the Sendspin server
```

To add a second device: flash it, run the bootstrap with a different hostname (e.g. `kitchen`), and commit a `host_vars/kitchen.yml` containing at least `sendspin_hostname: kitchen` plus any overrides.

### Useful variables

| Variable | Default | Default lives in | Purpose |
|----------|---------|------------------|---------|
| `sendspin_hostname` | `livingroom` | `group_vars/all.yml` | Enforced OS hostname for this device |
| `sendspin_name` | `{{ sendspin_hostname }}` | `group_vars/all.yml` | Client name advertised to the server |
| `sendspin_server_url` | example | `group_vars/all.yml` | Server URL in `settings-tui.json` — set the real URL here |
| `hifi_shield2_audio_device` | `""` (auto) | `hifi_shield2` role | Non-empty value pins the ALSA device, e.g. `hw:0,1` for optical |
| `hifi_shield2_enable_spdif` | `true` | `hifi_shield2` role | Adds the SPDIF (optical) PCM to the sound card |
| `sendspin_allow_reboot` | `true` | `group_vars/all.yml` | Let the play reboot when the overlay changes |
| `sendspin_use_hardware_volume` | `false` | `group_vars/all.yml` | Software volume only (set `true` later to control DAC/ALSA volume from the server) |
| `sendspin_visualizer` | `true` | `group_vars/all.yml` | Render the visualizer on launch |
| `sendspin_pull_interval_minutes` | `30` | `group_vars/all.yml` | Convergence timer interval |

Commit and push any change; devices pick it up on the next pull run. Note that `settings-tui.json` is fully managed, so runtime-saved settings (like volume) reset to defaults on each convergence run.

## Day-to-day operations

```bash
sudo /usr/local/bin/sendspin-pull.sh                  # force a convergence run now
journalctl -u sendspin-pull -n 100                   # what the last run did
journalctl -u sendspin-tui -f                        # TUI/player logs
journalctl -u hifishield2-init -n 50                 # mixer/overlay init
sudo -u sendspin ~/.local/bin/sendspin audio-devices list   # list ALSA devices
aplay -l                                             # raw card listing
sudo systemctl restart sendspin-tui                  # restart the player
```

If the shield's ALSA card is ever missing after a reboot, the convergence run **fails loudly** and prints `aplay -l`, `armbianEnv.txt`, and the audio-related `dmesg` lines into the pull log — check `journalctl -u sendspin-pull`.

## Outputs: RCA vs optical

The sound card exposes one PCM per output path (with `hifi_shield2_enable_spdif: true`):

- `hw:0,0` — I2S via the PCM5242 DAC: the **RCA** jacks (default, auto-selected)
- `hw:0,1` — SPDIF passthrough: the **optical** jack

Both jacks carry the same stream whenever sendspin plays, but you must set `hifi_shield2_audio_device: hw:0,1` (or the matching PortAudio index) in `group_vars` if your amplifier is on optical, since sendspin drives one ALSA device at a time. Volume is applied in software by default, so it works on both outputs.

## The HiFi Shield 2 overlay

The vendor instructions in `References/HiFi Shield 2 Setup.md` target Hardkernel's 3.14 vendor kernel, which does not exist on Armbian's mainline kernel. `roles/hifi_shield2/templates/hifishield2.dts.j2` replaces that approach with a device-tree overlay that:

- disables the stock HDMI-only `gx-sound-card`,
- instantiates the PCM5242 codec at I2C address `0x4c` on `i2c_A` (supported by the mainline `pcm512x` driver),
- defines a new `gx-sound-card` with an I2S front-end/backend pair wired to the codec, and (optionally) an SPDIF pair wired to a `spdif-dit` for the optical jack,
- muxes the AO-bank header pins (I2S + SPDIF) via `pinctrl` on the `aiu` node, the same pattern mainline uses on the WeTek Play2.

If the card fails to appear, the first knob to try is `hifi_shield2_enable_spdif: false` (drops the SPDIF links in case the `snd-soc-spdif-tx` module is unavailable), followed by checking the diagnostics block in the pull log.

## Console notes

- The TUI runs on the Linux console (`tty1`) with `TERM=linux`, so colors are limited to the 16-color palette — the visualizer renders, but with reduced color depth.
- Console blanking is disabled via `consoleblank=0` on the kernel command line (managed in `/boot/armbianEnv.txt`), so the display never sleeps.
- A USB keyboard plugged into the C2 controls the TUI directly (space, arrows, `m`, `g`, `v`, `q`) — useful later if you add a wireless keyboard/remote combo.
- With a keyboard you can also switch to other consoles with `Ctrl+Alt+F2` to get a normal login prompt.

## Layout

```
local.yml                  ansible-pull entry point (runs all roles)
group_vars/all.yml         shared defaults and knobs
host_vars/<hostname>.yml   per-device overrides
roles/
  hostname                 enforce sendspin_hostname
  ansible_pull_timer       self-management: wrapper + systemd timer running ansible-pull
  packages                 apt dependencies (portaudio, openblas, alsa-utils, dtc)
  hifi_shield2             DT overlay install/compile/activate, reboot, card detect,
                           audio device resolution, mixer defaults oneshot
  sendspin_user            dedicated system user (audio group)
  uv                       uv installer (as the sendspin user)
  sendspin                 uv tool install/upgrade sendspin
  sendspin_config          fully managed settings-tui.json
  sendspin_tui_service     TUI on tty1: getty mask, consoleblank=0, restart policy
References/                upstream docs kept for reference
```
