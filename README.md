# Vinyl Record Player (Turntable) to Music Assistant streaming guide

A step-by-step guide for one way to achieve music streaming from record players, USB turntables, or other analog sources via local URL to Music Assistant (MA), Sonos or Pi Musicbox, et al on a Raspberry Pi.

So you can use this solution regardless if your vinyl record player / LP turntable (or other audio source) includes a USB interface output port with built-in USB audio codec or if you need to buy and use a stand-alone USB Audio Device for external analogue-to-digital conversion.

## Icecast and Darkice based solution on Raspberry Pi 3 with Raspbian Jessie Lite

These solutions utilize a USB Audio Class 2.0 pipeline that can support high-definition audio formats up to 192KHz and 32bits using a standard digital audio interface.

To facilitate streaming audio from an analog vinyl record player and/or LP turntable with a built-in USB port for audio output you are going to have to have a few things:

1. USB Audio Device or a record player/turntable with a built-in USB audio codec output:

If you already own a record player/turntable that only has analog audio output (.i.e. it does not have a embedded USB audio codec output) then the easiest option is to use an external stand-alone USB Audio Device for analogue-to-digital conversion.

USB Audio Device adds the analog line-in interface ports and ADC (analogue-to-digital converter) that is needed for capturing the raw audio from your analog audio source (e.g. vinyl record player/turntable or cassette player) and make the analog audio stream available for streaming.

- [Behringer UFO202](https://www.behringer.com/product.html?modelCode=P0484) (with pre-amp)
- [Behringer UCA222](https://www.behringer.com/product.html?modelCode=0805-AAG) (without pre-amp, replaces the UCA202)
- [Behringer UCA202](https://www.behringer.com/behringer/product?modelCode=0805-AAC) (without pre-amp)
- [ART USB Phono Plus](https://artproaudio.com/product/usb-phono-plus-project-series/) (a standalone pre-amp with USB interface that needs external power-suppy).
- [IK Multimedia iRig Stream](https://www.ikmultimedia.com/products/irigstream/) (without pre-amp)
- [IK Multimedia iRig Stream Pro](https://www.dlxmusic.se/produkter/studio/ljudkort/externa/ik-irig-stream-pro) (with pre-amp)
- Another option as ADC instead of a USB Audio Device that should provide the same function but have not been tested here are HiFiBerry's ANALOG INPUT products like "HIFIBERRY DAC+ ADC PRO", "HIFIBERRY DAC2 ADC PRO", or "HIFIBERRY DAC+ ADC" as input. As a bonus those should provide a very clean and almost the look of an all-in-one commercial appliance:
  - https://www.hifiberry.com/blog/need-some-input/
    - https://www.hifiberry.com/shop/boards/hifiberry-dac2-adc-pro/
    - https://www.hifiberry.com/shop/boards/hifiberry-dac-adc-pro/
    - https://www.hifiberry.com/shop/boards/hifiberry-dac-adc/

Note! If your record player/turntable does not have a built-in pre-amp for analog output then you either need to buy specifically a USB Audio Device with pre-amp (or a seperate phono preamp to put inline) as otherwise you will not get a signal that has been amplified enough to allow good digitalization.

The alternative to above is to simply buy and use a "USB turntable" which is a vinyl record player that already has embedded USB audio codec output, like example one of these:

- Audio-Technica has several various different models of USB turntables:
  - Audio-Technica AT-LP5X
  - Audio-Technica AT-LP60XUSB
  - Audio-Technica AT-LP120XBT-USB
  - Audio-Technica AT-LP120XUSB
  - [Audio-Technica AT-LP60-USB](http://amzn.to/2dSVrGz)
  - [Audio-Technica AT-LP120-USB](http://amzn.to/2drytFC)
- [ION Audio Classic LP | 3-Speed USB](http://amzn.to/2e3piLt)
- [Sony PSLX300USB](http://amzn.to/2dW2Xm7)

2. Raspberry Pi 3 (or older, but why?) with accessories like an SD-card and power-supply:

- [Raspberry Pi 3](http://amzn.to/2eixlms)

3. Operating system:

Use [Raspbian Jessie Lite](https://www.raspberrypi.org/downloads/raspbian/) and follow the installtion instructions below.

## Install raspbian
Grab [Pi Filler](http://ivanx.com/raspberrypi/) to write the image file (.img) to your [2GB or larger SD card](http://amzn.to/2dVXFXR).

## Headless wi-fi setup (via macOS, optional)

Download [VirtualBox](https://www.virtualbox.org) and the [VirtualBox Extension Pack](https://www.virtualbox.org/wiki/Downloads) (needed for USB 2/3 SD card readers).

Download an [Ubuntu VirtualBox image](http://www.osboxes.org/ubuntu/) so we can access the EXT4 filesystem on the RPi boot card.

Load up the image, insert the microSD card, select the card reader from the VirtualBox USB menu bar icon, and edit the `/etc/wpa_supplicant/wpa_supplicant.conf` file. Add a network entry (or several) for your wi-fi network:

```
network={
  ssid="MyWiFiNetwork"
  psk="mywifinetworkpassword"
}
```

Eject the SD card reader from Linux and your Mac, and put the card in the RPi.

Power up your RPi, wait a minute or so, and now try sshing into the box with `ssh pi@raspberrypi.local` and the password `raspberry`

## Dependencies
Change the Raspbian source repo
```
sudo nano /etc/apt/sources.list
```
change this all to (rasbian jessie is now legacy)
```
deb http://legacy.raspbian.org/raspbian/ jessie main contrib non-free rpi
# Uncomment line below then 'apt-get update' to enable 'apt-get source'
deb-src http://legacy.raspbian.org/raspbian/ jessie main contrib non-free rpi
```
Update your Raspbian install:
```
sudo apt-get update
```

Set your Raspbian system hostname by editing `/etc/hostname` and change `raspberrypi` to:
```
vinyl
```
and also the line in `/etc/hosts` from raspberrypi to:
```
127.0.1.1       vinyl
```

Then install a bunch of needed packages:
```
sudo apt-get -y install aptitude apt-utils sudo unzip autoconf libtool libtool-bin checkinstall libssl-dev libasound2-dev libmp3lame-dev libpulse-dev alsa-utils avahi-daemon darkice
```
We will install the darkice package, but compile it later to add AAC+ support

## Compiling libaacplus
```
wget http://tipok.org.ua/downloads/media/aacplus/libaacplus/libaacplus-2.0.2.tar.gz
tar -xzf libaacplus-2.0.2.tar.gz
cd libaacplus-2.0.2
./autogen.sh --with-parameter-expansion-string-replace-capable-shell=/bin/bash --host=arm-unknown-linux-gnueabi --enable-static
make
sudo make install
```

## Get darkice source
```
cd ~
mkdir src
cd src
apt-get source darkice
cd darkice-1.2
```

## Compiling darkice

```
./configure --with-aacplus --with-aacplus-prefix=/usr/local --with-pulseaudio --with-pulseaudio-prefix=/usr/lib/arm-linux-gnueabihf --with-lame --with-lame-prefix=/usr/lib/arm-linux-gnueabihf --with-alsa --with-alsa-prefix=/usr/lib/arm-linux-gnueabihf --with-jack --with-jack-prefix=/usr/lib/arm-linux-gnueabihf
make
sudo make install
```

## Configure /etc/darkice.cfg
```
# this section describes general aspects of the live streaming session
[general]
duration        = 0         # duration of encoding, in seconds. 0 means forever
bufferSecs      = 1         # size of internal slip buffer, in seconds
reconnect       = yes       # reconnect to the server(s) if disconnected
realtime        = yes       # run the encoder with POSIX realtime priority
rtprio          = 3         # scheduling priority for the realtime threads

# this section describes the audio input that will be streamed
[input]
device          = hw:1,0    # OSS DSP soundcard device for the audio input
sampleRate      = 48000     # other settings have crackling audo, esp. 44100
bitsPerSample   = 16        # bits per sample. try 16
channel         = 2         # channels. 1 = mono, 2 = stereo

# this section describes a streaming connection to an IceCast2 server
# there may be up to 8 of these sections, named [icecast2-0] ... [icecast2-7]
# these can be mixed with [icecast-x] and [shoutcast-x] sections
[icecast2-0]
bitrateMode     = cbr
format          = mp3
# format          = aacp
bitrate         = 320
# bitrate         = 64
server          = vinyl
port            = 8000
password        = vinyl   # or whatever you set your icecast2 password to
mountPoint      = listen
name            = Vinyl
description     = DarkIce on Raspberry Pi
url             = http://vinyl
genre           = vinyl
public          = no
localDumpFile   = recording.m4a
```

## icecast2
```
sudo aptitude install icecast2
```
For the hostname, use `vinyl`, and for both *hackme* passwords, use `vinyl`

Then, for the admin password, set it to `vinyl` as well.

Note: heaven forbid you mess up the icecast2 text GUI config... you'll need to run
```
sudo apt-get autoremove icecast2
sudo apt-get purge icecast2
```
and then reinstall it
```
sudo aptitude install icecast2
```
to get that crappy GUI back... unless there's an easier, undocumented way? and even then, where are the icecast.xml config files? not in /etc/icecast2/ ...

## Disable the icecast2 burst-on-connect
Edit `/etc/icecast2/icecast.xml` and set burst-on-connect to 0 to lower latency on your local network:
```
<burst-on-connect>0</burst-on-connect>
```

## Autostart darkice
These are from an Ubuntu install and don't exactly match the startup script, but they are close enough and do solve the startup problem

### 1
In /etc/init.d/darkice find:
```
DAEMON=/usr/bin/$NAME
```
and change it to the AAC+ complied version:
```
DAEMON=/usr/local/bin/$NAME
```

### 2
In /etc/init.d/darkice find:
```
start-stop-daemon --start --quiet --pidfile $PIDFILE \
```
and replace it with:
```
start-stop-daemon --start --quiet -m --pidfile $PIDFILE \
```

### 3
In /etc/init.d/darkice find:
```
stop_server() {
# Stop the process using the wrapper
        start-stop-daemon --stop --quiet --pidfile $PIDFILE \
            --exec $DAEMON
        errcode=$?
```
add after (with the new line):
```
    rm $PIDFILE
```

### 4
In /etc/init.d/darkice find:
```
running() {
# Check if the process is running looking at /proc
# (works for all users)
```
add after (with the new line):
```
    sleep 1
```

### 5
In /etc/default/darkice check that you have
```
RUN=yes
```

### 6
```
sudo systemctl daemon-reload
```

### 7
Add default user nobody to the audio group (in my case, to work with ALSA):
```
sudo adduser nobody audio
```

### 8
Fix upstart problem (it seems Darkice is trying to start on boot too early):
```
sudo update-rc.d -f darkice remove
sudo update-rc.d darkice defaults 99
```

## Reboot and connect to your USB turntable
It should work now, so connect your streaming client up to (http://vinyl.local:8000/listen.m3u) and put on a record.

On Music-Assistant, add the URL to your radio station through the GUI.

On Sonos, add your streaming turntable URL (http://vinyl.local:8000/listen.m3u) by [adding a custom Internet radio station](https://sonos.custhelp.com/app/answers/detail/a_id/264/~/how-to-add-an-internet-radio-station-to-sonos).

On Pi Musicbox, add the URL to your `/boot/config/radiostations.js` file or use the GUI.

Or switch to [Volumio](https://volumio.org).

## Icecast2 admin
is located at (http://vinyl.local:8000) and is good for checking the status of connected clients

## RPi temp
Check the temp of your RPi 3 with
```
/opt/vc/bin/vcgencmd measure_temp
```
if you're running without a heatsink, best to keep it below 70C

## More info
See [this forum](http://ubuntuforums.org/showthread.php?t=2183222)

## Final note
While AAC+ is neat, on a local network you might as well stream 320Kbps MP3 for better sound quality, or if you're so inclined, uncompressed WAV
