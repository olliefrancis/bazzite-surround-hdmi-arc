# Getting 5.1 Surround Sound Working on Bazzite over HDMI ARC

**Platform:** Bazzite Linux  
**Goal:** Dolby Digital AC3 5.1 audio output over HDMI ARC to a TV/soundbar (e.g. Samsung TV + Sonos)  
**Status:** Working as of June 2026

---

## The Problem

Most TVs (Samsung included) don't support LPCM 5.1 over HDMI ARC. They only accept PCM 2.0 or Dolby Digital (AC3). Linux doesn't transcode to AC3 by default, so surround sound just doesn't arrive at your receiver.

The fix is to create a virtual audio device that transcodes audio to AC3 before sending it out over HDMI. The catch is that most guides assume a traditional PulseAudio setup, but Bazzite uses PipeWire - so there's an extra step that most tutorials miss.

---

## What You'll Need

- Bazzite (tested on a system with AMD/ATI HDMI audio)
- Your TV connected via HDMI ARC
- A terminal

---

## Step 1: Install the AC3 encoder plugin

Bazzite is an immutable OS, so software installs go through `rpm-ostree`:

```bash
sudo rpm-ostree install alsa-plugins-a52
```

Then reboot for the change to take effect.

---

## Step 2: Find your HDMI audio device

Run:

```bash
aplay -l
```

You're looking for the entry that corresponds to your TV. It'll look something like:

```
card 1: HDMI [HDA ATI HDMI], device 10: HDMI 4 [SAMSUNG]
```

Note down the **card number** and **device number**. In this example: card `1`, device `10`.

---

## Step 3: Install PulseAudio Volume Control

Install the **PulseAudio Volume Control** flatpak from the Bazzite app store (search for "pavucontrol").

Open it, go to the **Configuration** tab, and **disable** the HDMI output for your TV. This releases the device so our custom setup can take control of it.

---

## Step 4: Create the ALSA config

This tells ALSA how to encode audio as AC3 and send it to your specific HDMI output.

```bash
nano ~/.asoundrc
```

Paste the following, replacing `hw:HDMI,10` with your actual card and device numbers from Step 2:

```
# Raw AC3 encoder - strict about input format
pcm.a52_raw {
    type a52
    rate 48000
    bitrate 640
    channels 6
    format S16_LE
    slavepcm "hw:HDMI,10"
}

# Safe wrapper - converts app audio to the format the encoder needs
pcm.a52_safe {
    type plug
    slave.pcm "a52_raw"
    hint {
        show on
        description "TV (AC3 Surround)"
    }
}
```

The bitrate is set to 640kbit, which is the maximum for HDMI ARC.

---

## Step 5: Test it

```bash
speaker-test -D a52_safe -c 6 -r 48000
```

You should hear the speaker test tones coming through your surround system. If this works, carry on. If not, double-check your card/device numbers in Step 2.

---

## Step 6: Create the PipeWire sink (the bit most guides miss)

Most tutorials tell you to create a systemd service that runs `pactl load-module module-alsa-sink`. **This doesn't work on Bazzite** because PipeWire's PulseAudio compatibility layer doesn't support that module. You'll get `Failure: Unknown error code` and the sink will never appear after a reboot.

The fix is to create the sink using PipeWire's own native config instead.

```bash
mkdir -p ~/.config/pipewire/pipewire.conf.d
nano ~/.config/pipewire/pipewire.conf.d/ac3-sink.conf
```

Paste this in:

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

---

## Step 7: Create a keepalive service (optional but recommended)

Without this, your receiver may click or go silent during quiet moments because it thinks the audio stream has ended.

```bash
nano ~/.config/systemd/user/ac3-keepalive.service
```

```ini
[Unit]
Description=Keep AC3 Output Alive with Silence
After=pipewire.service

[Service]
Type=simple
ExecStart=/usr/bin/paplay --raw --format=s16le --rate=48000 --channels=2 --device=TV_AC3 --property=media.role=background --latency-msec=2000 /dev/zero
Restart=always
RestartSec=5

[Install]
WantedBy=default.target
```

Enable it:

```bash
systemctl --user daemon-reload
systemctl --user enable --now ac3-keepalive.service
```

---

## Step 8: Restart PipeWire and check

```bash
systemctl --user restart pipewire pipewire-pulse wireplumber
```

Then verify the sink exists:

```bash
pactl list short sinks
```

You should see `TV_AC3` in the list. 

---

## Step 9: Set TV_AC3 as your default output

In your audio settings (or in PulseAudio Volume Control under the **Output Devices** tab), set **TV Surround 5.1** as your default output. You can also select it per-application in the **Playback** tab.

---

## Reboot and confirm

Reboot your machine and run `pactl list short sinks` again to confirm `TV_AC3` is still there. It should be - PipeWire creates it automatically at startup from the config file you created in Step 6.

---

## Troubleshooting

**TV_AC3 doesn't appear after reboot**  
Check that the file `~/.config/pipewire/pipewire.conf.d/ac3-sink.conf` exists and the syntax is correct. Then restart PipeWire with the command in Step 8.

**Speaker test works but no surround in games/movies**  
Make sure TV_AC3 is selected as the output device, either globally or per-application.

**ALSA card number changed after reboot**  
This is a known issue. Run `aplay -l` again to find the new card/device numbers and update `~/.asoundrc` accordingly.

**LG TV users**  
There are reports of loud white noise occurring occasionally with LG TVs. Proceed with caution.

---

## Credits

Based on the original guide by [basso](https://gist.github.com/basso/04cbdc9cad5629f2ae83a941875c4ad5), with the PipeWire fix added for Bazzite compatibility.
