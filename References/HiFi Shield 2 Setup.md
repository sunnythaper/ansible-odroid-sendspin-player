## HiFi-shield2

The Hifi-Shield 2 provides stereo RCA connectors to connect to external amplifiers and other audio equipment. An S/PDIF Optical output is also available. Via TI's high-end **PCM5242** DAC chip(known as Burr-Brown), it supports 16, 24, 32 bit audio formats with minimal THD+N ratio (0.002%) and ideal dynamics (116dB+), plus amazing sampling rates of 384kHz with odroid-c1. (up to 192KHz with odroid-c2) Volume control via I2C interface is great feature to keep the great audio quality with various audio output volume.

- Assemble the C2 with the Hifi-Shield2, attach a USB keyboard, USB mouse and HDMI monitor, then power up the system.
- Update the System: **sudo apt-get update && sudo apt-get dist-upgrade**
- To ensure that the Hifi-Shield2 kernel modules are loaded on Terminal.

[target](https://wiki.odroid.com/_export/code/accessory/sound/hifi_shield?codeblock=6 "Download Snippet")

```bash
odroid@odroid64:~$ aplay -l
**** List of PLAYBACK Hardware Devices ****
card 0: ODROIDHDMI [ODROID-HDMI], device 0: I2S dit-hifi-0 []
  Subdevices: 1/1
  Subdevice #0: subdevice #0
odroid@odroid64:~$ su
Password:  /* root password is "odroid" */
root@odroid64:/home/odroid# modprobe aml_i2c
root@odroid64:/home/odroid# modprobe snd_soc_pcm512x
root@odroid64:/home/odroid# modprobe snd_soc_pcm512x_i2c
root@odroid64:/home/odroid# echo pcm5242 0x4c > /sys/class/i2c-adapter/i2c-1/new_device
root@odroid64:/home/odroid# modprobe snd-soc-odroid-dac2
root@odroid64:/home/odroid# aplay -l
**** List of PLAYBACK Hardware Devices ***
card 0: ODROIDHDMI [ODROID-HDMI], device 0: I2S dit-hifi-0 []
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: ODROIDDAC2 [ODROID-DAC2], device 0: I2S pcm512x-hifi-0 []
  Subdevices: 0/1
  Subdevice #0: subdevice #0
root@odroid64:/home/odroid#
```

- To test, connect an amplifier to the RCA out jack on the Hifi-Shield2, and then run:
- Applications → Sound & Video → Sound → Hardware Tab & Output Tab select “ODROID-DAC2”
- If you want to use the **Optical output** select “ODROID-HDMI”.

**You want to add the Hifi-Shield2 sound\_card on ODROID-C2 at boot by editing /etc/rc.local.**

[target](https://wiki.odroid.com/_export/code/accessory/sound/hifi_shield?codeblock=7 "Download Snippet")

```bash
odroid@odroid64:~$ sudo nano /etc/rc.local
if [ -f /aafirstboot ]; then /aafirstboot start ; fi

echo Loading HiFi-Shield2 module
modprobe aml_i2c
modprobe snd-soc-pcm512x
modprobe snd-soc-pcm512x-i2c
echo pcm5242 0x4c > /sys/class/i2c-adapter/i2c-1/new_device
modprobe snd-soc-odroid-dac2

amixer -c 1 sset Analogue 0
amixer -c 1 sset Digital 201
amixer -c 1 sset Deemphasis on
echo Done.

exit 0

^G Get Help  ^O Write Out ^W Where Is  ^K Cut Text  ^J Justify   ^C Cur Pos
^X Exit      ^R Read File ^\ Replace   ^U Uncut Text^T To Linter ^_ Go To Line
```

- Add the following at the bottom of the page above exit 0. ^x exit and save the file. Reboot your ODROID-C2.

**Create /etc/asound.conf with the following content:**

```js
pcm.!default  {
 type hw card 1
}
ctl.!default {
 type hw card 1
}
```

Reboot again.

**Check, if the sound card is enabled with “aplay -l”:**

[target](https://wiki.odroid.com/_export/code/accessory/sound/hifi_shield?codeblock=9 "Download Snippet")

```bash
odroid@odroid64:~$ aplay -l
**** List of PLAYBACK Hardware Devices ****
card 0: ODROIDHDMI [ODROID-HDMI], device 0: I2S dit-hifi-0 []
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: ODROIDDAC2 [ODROID-DAC2], device 0: I2S pcm512x-hifi-0 []
  Subdevices: 1/1
  Subdevice #0: subdevice #0
odroid@odroid64:~$
```

Hifi Shield and Hifi Shield 2 sound cards cannot be activated at the same time. Because they share I2S port.
Please use only one of them.

If above guide is not comfortable for you, try this alternative easy solution made by Fourdee ([DietPi](https://wiki.odroid.com/odroid_c2/dietpi "odroid_c2:dietpi") OS image author).
[http://forum.odroid.com/viewtopic.php?f=118&t=24415](http://forum.odroid.com/viewtopic.php?f=118&t=24415 "http://forum.odroid.com/viewtopic.php?f=118&t=24415")

The volume level control is available only on the RCA line-out output of [HiFi](https://wiki.odroid.com/accessory/sound/hifi "accessory:sound:hifi") -Shield 2.
The SPDIF optical output level is always maximum because it is a simple I2S bypass.
