---
layout: post
title: Building a DIY Voice Satellite for Home Assistant
date: 2024-08-19 21:52
tag: smart_home
---

## Intro

In the realm of home automation, there are few platforms as flexible or capable as [Home Assistant](https://www.home-assistant.io/). In this post, I'll go through the steps I used to setup a voice satellite (think Amazon Echo Dot) with a custom local wake word and also act as a target for automated text-to-speech notifications. These steps are HEAVILY influenced from the amazing [Slacker Labs](https://mastodon.online/@slackerlabs) and his write-up [here](https://www.slacker-labs.com/setup-a-raspberry-pi-zero-2-w-as-a-wyoming-satellite/). Please check out his [YouTube](https://www.youtube.com/c/SlackerLabs) for more great content.

### Prerequisite Hardware:
- Raspberry Pi Zero 2 with microSD card
- ReSpeaker 2-Mic HAT
- 3 watt 4 Ohm speaker, preferably with a JST connector

## Creating a Custom Wake Word

As cool as saying "Hey Jarvis" is, I wanted to let my nerd flag fly a bit more. See, I grew up on episodes Star Trek: The Next Generation, therefore I wanted my voice assistant to be named Geordi, after the chief engineer, Geordi La Forge, of the USS Enterprise (NCC-1701-D). 

Luckily, Home Assistant has a great write up [here](https://www.home-assistant.io/voice_control/create_wake_word/#to-create-your-own-wake-word) which works wonderfully. Once your custom wake word is generated and you have the .tflte file, you can move on to configuring the Raspberry Pi.

## Configure the Raspberry Pi Zero 2

Follow the steps in Slacker Lab's write up to install the Raspberry Pi Lite OS. Make sure to assign the hostname and user to be something recognizable and also set your region specific information. Raspberry Pi's typically default to en-GB settings since they're a UK based company.

Note: Do not connect the ReSpeaker HAT to the Pi since we still need to install the drivers.

Once connected to the Pi over SSH, run the following commands:

Update the package databases
```
sudo apt-get update
```

Install Git and Python Virtual Environment packages
```
sudo apt-get install --no-recommends git python3-venv
```

Clone the Wyoming-Satellite Files
```
git clone https://github.com/rhasspy/wyoming-satellite.git
```

Navigate to the Wyoming-Satellite Directory
```
cd wyoming-satellite
```

Install the ReSpeaker drivers
```
sudo bash etc/install-respeaker-drivers.sh
```

Assuming no errors occur, reboot the Pi so the drivers are loaded. You can attach the ReSpeaker HAT now but be sure to do so while while the Pi is powered off.

### Install and Configure the Wyoming Satellite Service

Reconnect to the Pi over SSH and enter the following commands:

Navigate to the Wyoming-Satellite Directory
```
cd wyoming-satellite
```

Install Dependencies and Python Environment
```
python3 -m venv .venv
.venv/bin/pip3 install --upgrade pip
.venv/bin/pip3 install --upgrade wheel setuptools
.venv/bin/pip3 install \
  -f 'https://synesthesiam.github.io/prebuilt-apps/' \
  -r requirements.txt \
  -r requirements_audio_enhancement.txt \
  -r requirements_vad.txt
```

Verify ReSpeaker HAT is Detected
```
arecord -L
```
You should see a line starting with `plughw:CARD=seeed2micvoicec,DEV=0`. This indicates that the HAT is successfully identified.

Create the Wyoming-Satellite Service
```
sudo systemctl edit --force --full wyoming-satellite.service
```

Copy and Paste the Service Configuration.
NOTE: If you're not using the `pi` user, be sure to replace that values with your username in the `ExecStart` and the `WorkingDirectory` lines. Also, change `my-satellite` to whatever you want the device to be called.
```
[Unit]
Description=Wyoming Satellite
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
ExecStart=/home/pi/wyoming-satellite/script/run --name 'my satellite' --uri 'tcp://0.0.0.0:10700' --mic-command 'arecord -D plughw:CARD=seeed2micvoicec,DEV=0 -r 16000 -c 1 -f S16_LE -t raw' --snd-command 'aplay -D plughw:CARD=seeed2micvoicec,DEV=0 -r 22050 -c 1 -f S16_LE -t raw' --mic-auto-gain 5 --mic-noise-suppression 2
WorkingDirectory=/home/pi/wyoming-satellite
Restart=always
RestartSec=1

[Install]
WantedBy=default.target
```
To save this file and exit, press `CTRL-O` and then `CTRL-X`.

Enable the Wyoming-Satellite Service
```
sudo systemctl enable --now wyoming-satellite.service
```

Check that the Service is Running
```
journalctl -u wyoming-satellite.service -f
```
or
```
sudo systemctl status wyoming-satellite
```

