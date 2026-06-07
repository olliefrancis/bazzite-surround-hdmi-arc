# Getting 5.1 Surround Sound Working over HDMI ARC on Bazzite

*Bazzite Linux / Audio / Guide*

Dolby Digital AC3 transcoding on Bazzite with PipeWire — including the fix that most guides miss.

**Tags:** Bazzite · PipeWire · HDMI ARC · AC3 / Dolby Digital · June 2026

**Web guide:** [bazzite-surround-hdmi-arc.olliefrancis.com](https://bazzite-surround-hdmi-arc.olliefrancis.com/) · **[View on GitHub](https://github.com/olliefrancis/bazzite-surround-hdmi-arc)**

---

## Contents

1. [Install the AC3 encoder plugin](#step-1-install-the-ac3-encoder-plugin)
2. [Find your HDMI audio device](#step-2-find-your-hdmi-audio-device)
3. [Release the HDMI device in PulseAudio Volume Control](#step-3-release-the-hdmi-device-in-pulseaudio-volume-control)
4. [Create the ALSA config](#step-4-create-the-alsa-config)
5. [Test the encoder](#step-5-test-the-encoder)
6. [Create the PipeWire sink](#step-6-create-the-pipewire-sink)
7. [Create a keepalive service](#step-7-create-a-keepalive-service-recommended)
8. [Restart PipeWire and verify](#step-8-restart-pipewire-and-verify)
9. [Reboot and confirm](#step-9-reboot-and-confirm)
10. [Troubleshooting](#troubleshooting)

---

Most TVs — Samsung included — don't support LPCM 5.1 over HDMI ARC. They only accept PCM 2.0 or Dolby Digital (AC3). Linux doesn't transcode to AC3 by default, so surround sound just doesn't arrive at your receiver even if everything looks correctly configured.

The fix is to create a virtual audio device that encodes your audio as AC3 before it goes out over HDMI. There's a good existing guide for this, but it assumes traditional PulseAudio. Bazzite uses PipeWire, and there's one step that breaks silently — your sink appears to work but vanishes after every reboot. This guide covers the full setup including that fix.

> **Before you start.** Several steps use the **Terminal** — a text-based app where you type or paste commands and press **Enter** to run them. On Bazzite, open it from your app menu (search for **Terminal** or **Konsole**). To paste a command, copy it from this guide, click inside the Terminal window, and press **Ctrl+Shift+V** (or right-click and choose Paste). Some commands start with `sudo`, which means you'll be asked for your login password — type it and press Enter. Nothing will appear on screen as you type the password; that's normal.

---

## Step 1: Install the AC3 encoder plugin

Bazzite is an immutable OS, so software installs go through `rpm-ostree` rather than a standard package manager.

1. Open the **Terminal**.
2. Copy the command below, paste it into the Terminal, and press **Enter**.
3. Enter your password if prompted, then wait for the install to finish.
4. **Reboot your PC** — the plugin won't be available until you do.

```bash
sudo rpm-ostree install alsa-plugins-a52
```

---

## Step 2: Find your HDMI audio device

You need to find which card and device number Linux uses for your TV's HDMI output.

1. Open the **Terminal**.
2. Paste the command below and press **Enter**.
3. Read through the output and look for a line that mentions your TV — it'll usually show the manufacturer name (e.g. Samsung).
4. Note down the **card number** and **device number** from that line.

```bash
aplay -l
```

Example output — yours will differ:

```
# Example output - yours will differ
card 1: HDMI [HDA ATI HDMI], device 10: HDMI 4 [SAMSUNG]
```

In this example: card `1`, device `10`. Keep these handy for the next step.

---

## Step 3: Release the HDMI device in PulseAudio Volume Control

This step uses a graphical app rather than the Terminal.

1. Open the **Bazzite app store** and install **PulseAudio Volume Control** (search for "pavucontrol").
2. Launch PulseAudio Volume Control from your app menu.
3. Click the **Configuration** tab.
4. Find the HDMI output connected to your TV and set it to **Off** or **Disabled**.

This releases the HDMI device so the custom AC3 setup can take control of it in the next steps.

---

## Step 4: Create the ALSA config

This creates a config file that tells ALSA how to encode audio as AC3 and route it to your HDMI output. You'll create the file in a text editor called **nano**, which opens inside the Terminal.

1. Open the **Terminal**.
2. Run the command below — this opens (or creates) the file `~/.asoundrc` in nano. `~` means your home folder.

```bash
nano ~/.asoundrc
```

3. Copy the config block below. In nano, paste it with **Ctrl+Shift+V** (or right-click → Paste).
4. Find the line `"hw:HDMI,10"` and change it to use your numbers from Step 2. For example, if your card was `1` and device was `10`, that line should read `"hw:1,10"`.
5. Save the file: press **Ctrl+O**, then **Enter** to confirm the filename.
6. Exit nano: press **Ctrl+X**.

```
# Raw AC3 encoder
pcm.a52_raw {
    type      a52
    rate      48000
    bitrate   640
    channels  6
    format    S16_LE
    slavepcm  "hw:HDMI,10"  # your card and device here
}

# Safe wrapper - handles format conversion for apps
pcm.a52_safe {
    type plug
    slave.pcm "a52_raw"
    hint {
        show on
        description "TV (AC3 Surround)"
    }
}
```

The bitrate of 640kbit is the maximum for HDMI ARC.

---

## Step 5: Test the encoder

Before going further, confirm that audio is actually reaching your surround system.

1. Open the **Terminal**.
2. Paste the command below and press **Enter**.
3. You should hear test tones from each speaker. Press **Ctrl+C** to stop the test when you're done.

```bash
speaker-test -D a52_safe -c 6 -r 48000
```

If you don't hear anything, go back and check your card and device numbers in Step 2.

---

## Step 6: Create the PipeWire sink

> **Other guides may still work on older versions of Bazzite.** The usual advice is to run `pactl load-module module-alsa-sink` via a systemd service. On current Bazzite this silently fails with *"Failure: Unknown error code"* because PipeWire's PulseAudio compatibility layer doesn't support that module. Your sink may appear to work in testing but vanishes after every reboot. The fix is to create the sink using PipeWire's own native config file instead.

1. Open the **Terminal**.
2. Run the first command below to create the folder (if it doesn't exist already).
3. Run the second command to open a new config file in nano.

```bash
mkdir -p ~/.config/pipewire/pipewire.conf.d
nano ~/.config/pipewire/pipewire.conf.d/ac3-sink.conf
```

4. Paste the config below into nano.
5. Save with **Ctrl+O**, then **Enter**.
6. Exit with **Ctrl+X**.

```
context.objects = [
  {
    factory = adapter
    args = {
      factory.name     = api.alsa.pcm.sink
      node.name        = TV_AC3
      node.description = "TV Surround 5.1"
      media.class      = Audio/Sink
      api.alsa.path    = a52_safe
      audio.channels   = 6
      audio.rate       = 48000
    }
  }
]
```

PipeWire will create the `TV_AC3` sink automatically at startup from this config file.

---

## Step 7: Create a keepalive service (recommended)

Without this, your receiver may click or go silent during quiet moments because it thinks the audio stream has ended. This service plays inaudible silence into the sink to keep it active.

1. Open the **Terminal**.
2. Run the command below to create a new service file in nano.

```bash
nano ~/.config/systemd/user/ac3-keepalive.service
```

3. Paste the service config below into nano.
4. Save with **Ctrl+O**, then **Enter**.
5. Exit with **Ctrl+X**.
6. Back at the Terminal prompt, paste and run the two commands below one at a time (press **Enter** after each).

```ini
[Unit]
Description=Keep AC3 Output Alive with Silence
After=pipewire.service

[Service]
Type=simple
ExecStart=/usr/bin/paplay --raw --format=s16le --rate=48000 \
  --channels=2 --device=TV_AC3 \
  --property=media.role=background \
  --latency-msec=2000 /dev/zero
Restart=always
RestartSec=5

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now ac3-keepalive.service
```

---

## Step 8: Restart PipeWire and verify

1. Open the **Terminal**.
2. Paste and run the command below to restart the audio system.

```bash
systemctl --user restart pipewire pipewire-pulse wireplumber
```

3. Then paste and run this command to list available audio outputs:

```bash
pactl list short sinks
```

You should see `TV_AC3` in the list. Set it as your default output in your audio settings, or select it per-application in PulseAudio Volume Control under the Playback tab.

---

## Step 9: Reboot and confirm

1. **Reboot your PC** using the normal shutdown/restart menu.
2. After logging back in, open the **Terminal** and run:

```bash
pactl list short sinks
```

`TV_AC3` should still be in the list — PipeWire creates it automatically from the config file on every startup.

**Done.** You should now have Dolby Digital 5.1 output over HDMI ARC that survives reboots.

---

## Troubleshooting

**TV_AC3 doesn't appear after reboot**  
Check that `~/.config/pipewire/pipewire.conf.d/ac3-sink.conf` exists and the syntax is correct. Then restart PipeWire with the command in Step 8.

**Speaker test works but no surround in games or movies**  
Make sure TV_AC3 is selected as the output device — either globally in sound settings or per-application in PulseAudio Volume Control.

**ALSA card number changed after reboot**  
This is a known issue with some systems. Run `aplay -l` again to find the new card and device numbers, then update `~/.asoundrc` accordingly.

**LG TV users**  
There are reports of loud white noise occurring occasionally with LG TVs from around 2017. Proceed with caution.

---

## Credits

Guide written by [Ollie Francis](https://olliefrancis.com). [View on GitHub](https://github.com/olliefrancis/bazzite-surround-hdmi-arc).

Based on the original guide by [basso on GitHub](https://gist.github.com/basso/04cbdc9cad5629f2ae83a941875c4ad5), with the PipeWire native sink fix added for Bazzite compatibility.

Also available as a web guide at [bazzite-surround-hdmi-arc.olliefrancis.com](https://bazzite-surround-hdmi-arc.olliefrancis.com/).
