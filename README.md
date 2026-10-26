# iMac CS8409 on Linux

Out-of-tree `snd-hda-codec-cs8409` module for Apple machines with a Cirrus Logic CS8409, plus a capture fix for the 2017 iMac.

On iMac18,1 / 18,2 / 18,3 the stock driver binds the ALSA capture stream to a dead line-in ADC. Recordings are a half-second electrical pop, then digital silence. The internal mic ADC is configured and then never attached to a PCM stream.

This tree forces that capture stream onto the internal-mic ADC (`spec->intmike_adc_nid`, `0x23` on these iMacs) unless it is already that node or the headset mic (`0x1a`).

Confirmed on an iMac18,2 (subsystem `0x106b0f00`), Omarchy, kernel `7.2.8-9-omarchy`. After this module loads:

```bash
arecord -D hw:0,0 -f S32_LE -r 44100 -c 2 -d 4 /tmp/mic.wav
```

records a real signal for the whole clip. `S16_LE` is not a supported format on this path. Use `S32_LE`, `S24_LE`, or `S24_3LE` at 44100 Hz.

## Where this comes from

- Codec logic is from Cirrus Logic and the Linux ALSA HDA drivers. SPDX headers in the sources still apply. GPL-2.0-or-later.
- Apple machine support comes from [davidjo/snd_hda_macbookpro](https://github.com/davidjo/snd_hda_macbookpro) and [egorenar/snd-hda-codec-cs8409](https://github.com/egorenar/snd-hda-codec-cs8409).
- The Linux 6.17+ HDA API port (`hda_codec_driver`, `snd_hda_gen_remove`) is [daviddoll/snd-hda-codec-cs8409](https://github.com/daviddoll/snd-hda-codec-cs8409) branch `kernel-6.17-hda-compat`.
- The capture-stream force is the change on top of that branch. The same failure mode was diagnosed on an iMac18,1 by [@stefanopatrizi](https://github.com/egorenar/snd-hda-codec-cs8409/issues/5).

The source comments already map these subsystem IDs to the iMac layout (internal mic pin `0x45`, ADC `0x23`):

| Subsystem | Model |
|-----------|--------|
| `0x106b0e00` | iMac18,1 |
| `0x106b0f00` | iMac18,2 |
| `0x106b1000` | iMac18,3 (also noted as iMac19,1) |

Only 18,2 was tested here.

## Requirements

- Kernel headers for the running kernel (`linux-headers-$(uname -r)`, or `linux-omarchy-headers` on Omarchy)
- `gcc`, `make`

Build from this directory. `uname -r` has to match the target kernel, or pass `KVER`.

## Build

```bash
make
```

Do not run a bare `make -C /lib/modules/$(uname -r)/build M=$PWD modules`. That skips `CFLAGS_MODULE`, and this tree expects `APPLE_CODECS`, `APPLE_PINSENSE_FIXUP`, and `CONFIG_SND_HDA_RECONFIG`.

Build for another installed kernel:

```bash
make KVER=7.2.8-9-omarchy
```

## Install

```bash
sudo make install
```

That copies `snd-hda-codec-cs8409.ko` to `/lib/modules/$(uname -r)/updates/` and runs `depmod`. `updates/` is preferred over the in-tree module.

Reboot to pick it up. This is not a DKMS package, so a kernel upgrade needs a rebuild.

To go back to the stock module, delete the file under `updates/` and reboot.

The module is unsigned. Loading it taints the kernel.

## Loading it without a reboot

Prefer a reboot. A live swap is easy to wedge, and this machine has been wedged by it.

- Stop whatever is holding the codec first (PipeWire, WirePlumber, anything recording). Stopping the services is not enough if their sockets are still active.
- Unbind the codec (`hdaudioC0D0` on the `snd_hda_codec_cs8409` driver). Do not PCI-remove or rebind the analog controller. That wedged audio here and did not come back until reboot.
- If `insmod` fails with `Unknown symbol` for `snd_hda_gen_*`, load `snd-hda-codec-generic` first, then insert this module.
- If unbind stays busy or does not finish, reboot. Do not keep rebinding the controller.

If capture then fails with an I/O error and the internal-mic ADC is still in D3:

```bash
hda-verb /dev/snd/hwC0D0 0x23 0x705 0x00
```

That forces the ADC to D0. It is not sticky across reboot. `hda-verb` is in `alsa-tools`.

## License

GPL-2.0-or-later. See `COPYING` and the SPDX headers in the sources.
