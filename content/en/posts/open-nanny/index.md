---
title: Open Nanny
date: 2025-04-01
author: Knowledge Escalation
description: Build a Secure Raspberry Pi Baby Monitor with Real-Time Streaming
isStarred: false
---



![OpenNannyFront](1.jpg)

{{< justify >}}
Secure, Smart, and Open: Building a Feature-Rich Raspberry Pi Baby Monitor with Real-Time Streaming, Environmental Monitoring, and Lullaby Playback for Modern Parents.
{{< /justify >}}
## Introduction

{{< justify >}}
Are you a tech-savvy parent looking for a secure, customizable, and affordable baby monitoring solution? Commercial baby monitors often come with limited features, security concerns, and high price tags. The **OpenNanny** project solves these problems by leveraging the versatility of Raspberry Pi to create a feature-rich baby monitor with capabilities that surpass many commercial options.

This DIY smart baby monitor combines real-time video streaming, environmental monitoring, and lullaby playback in one integrated system. Whether you're concerned about indoor air quality, want secure video monitoring, or need a reliable way to play soothing sounds for your baby, OpenNanny delivers these features with the added benefits of complete privacy control and expandability.

Even if you're new to Raspberry Pi projects, this step-by-step guide will help you build a professional-quality monitoring system that grows with your needs. The OpenNanny system consists of two main components.
{{< /justify >}}

1. [**OpenNannyApi**](https://github.com/knowledgeescalation/OpenNannyApi.git): The backend software that controls the Raspberry Pi hardware and provides a secure API for remote management.
2. [**OpenNannyApp**](https://github.com/knowledgeescalation/OpenNannyApp.git): An Android application that connects to your OpenNanny device for monitoring and control.

### OpenNannyApi Features:

- **Environmental Monitoring**: Track CO2 levels, temperature, and humidity in real time.
- **Power Management**: Monitor UPS status, including voltage, current, and battery percentage.
- **Video Streaming**: Enjoy live video feeds with night mode via WebRTC.
- **Audio Communication**: One-way audio streaming for seamless communication.
- **Music Player**: Play lullabies with MP3 playback capabilities.
- **Secure API**: Protect your data with JWT authentication and TLS encryption.
- **Data Storage**: Store time-series data using InfluxDB for analysis.

### OpenNannyApp Features:

- Real-time sensor data visualization (temperature, humidity, CO2 levels).
- Live video streaming from the Raspberry Pi camera.
- Music playback controls (play, pause, volume adjustment).
- Night light control for LED lighting.



<img src="main_activity.jpg" alt="main_activity" style="width:30%; height:auto;"
title=" This is a delicious bowl of ice cream.">  <img src="sensors_activity.jpg" alt="sensors_activity" style="width:30%; height:auto;">  <img src="video_activity.jpg" alt="video_activity" style="width:30%; height:auto;">
 

<img src="music_activity_directory.jpg" alt="music_activity_directory" style="width:30%; height:auto;">  <img src="music_activity_songs.jpg" alt="music_activity_songs" style="width:30%; height:auto;">  <img src="music_activity_player.jpg" alt="music_activity_player" style="width:30%; height:auto;">


---

## Hardware Requirements

Before starting this project, gather these components for a complete OpenNanny system.

| Component                                                                                                                 | Purpose                                      |
| :------------------------------------------------------------------------------------------------------------------------ | :------------------------------------------- |
| Raspberry Pi Zero 2 WH                                                                                                    | Main processing unit                         |
| SD Card with Raspbian                                                                                                     | Operating system storage (Raspbian)          |
| [DFRobot SEN0536](https://wiki.dfrobot.com/SKU_SEN0536_Gravity_SCD41_Infrared_CO2_Sensor#target_9) Sensor                 | CO2, temperature, and humidity monitoring    |
| [SPH0645LM4H](https://learn.adafruit.com/adafruit-i2s-mems-microphone-breakout/raspberry-pi-wiring-test) Microphones (x2) | Audio input                                  |
| [MAX98357A](https://learn.adafruit.com/adafruit-max98357-i2s-class-d-mono-amp/raspberry-pi-wiring) Amplifier            | Audio output processing\|                    |
| [speaker 1w 8 ohm](https://www.granvozchina.com/info/how-to-wire-speakers-in-parallel-series-and-98026087.html) (x2)      | Audio playback                               |
| EEPROM (24LC256)                                                                                                          | HAT identification (only 128 bytes needed)   |
| CSI Camera with IR Cut (OV5647)                                                                                           | Video streaming with night vision capability |
| 3W IR LEDs (x2)                                                                                                           | Enhanced night vision illumination           |
| NPN Transistors (2N2222) (x2)                                                                                             | IR LED control circuits                      |
| 1kΩ resistor (x2)                                                                                                         | IR LED control circuits                      |
| 4000mAh Battery                                                                                                           | Power supply                                 |
| [UPS HAT (Waveshare 19739)](https://www.waveshare.com/wiki/UPS_HAT_(C))                                                   | Power management and monitoring              |


## Getting Started: Basic Setup

Before diving into the hardware assembly, set up your Raspberry Pi with the required software environment:

1. Flash Raspbian OS to your SD card.
2. Boot your Raspberry Pi and complete initial setup (setting locale, connecting to WiFi, etc.).
3. Update your system packages:

```bash
sudo apt update && sudo apt upgrade -y
```

4. Clone the OpenNanny repository:

```bash
ssh USER_NAME@NANNY_IP
git clone https://github.com/knowledgeescalation/OpenNannyApi
```


## Environmental Sensor Installation

{{< justify >}} 
The environmental sensing capabilities of OpenNanny provide valuable information about your baby's room conditions.
{{< /justify >}} 

![OpenNannyTop](sensor_xi.jpg)

### Setting Up the CO2 and Temperature Sensor

{{< justify >}} 
The `DFRobot SEN0536` is a Gravity-compatible sensor that measures carbon dioxide levels, temperature, and humidity - all critical environmental factors for a nursery.
{{< /justify >}}


<img src="sensors.png" alt="esensors" style="width:70%; height:auto;">

See: https://wiki.dfrobot.com/SKU_SEN0536_Gravity_SCD41_Infrared_CO2_Sensor#target_9

1. Connect the sensor to the Raspberry Pi I2C pins:
    - Connect VCC to 3.3V
    - Connect GND to ground
    - Connect SDA to GPIO2 (SDA)
    - Connect SCL to GPIO3 (SCL)
2. Verify the connection using the I2C detection tool:

```bash
sudo i2cdetect -y 1
```

You should see the sensor at address `0x62` in the output table.
```bash
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:                         -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
40: -- -- -- 43 -- -- -- -- -- -- -- -- -- -- -- --  --> UPS HAT
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- 62 -- -- -- -- -- -- -- -- -- -- -- -- --  --> co2 sensor
70: -- -- -- -- -- -- -- --
```


3. Test the sensor functionality with the provided script:

```bash
cd ~/OpenNannyApi/sensors
python co2-test.py
```

If working correctly, you'll see output showing CO2 concentration, temperature, and humidity readings.

```bash
Sensor begin successfully!!!
The current temperature compensation value : 4.00 C
Set the current environment altitude : 130 m
Automatic calibration on!
Carbon dioxide concentration : 953 ppm
Environment temperature : 22.23 C
Relative humidity : 47.64 RH
```
See: https://github.com/knowledgeescalation/OpenNannyApi/blob/main/sensors/co2-test.py
### Installing the UPS Power Management System


<img src="ups.png" alt="ups" style="width:100%; height:auto;">

See: https://www.waveshare.com/wiki/UPS_HAT_(C).

The `UPS HAT` provides battery backup and power monitoring capabilities.

1. Attach the UPS HAT to your Raspberry Pi GPIO pins.
2. Secure with screws through the mounting holes.
3. Connect your 4000mAh battery to the UPS HAT.
4. Verify the connection with:

```bash
sudo i2cdetect -y 1
```

The UPS should appear at address `0x43`.

5. Test the UPS functionality:

```bash
python ups-test.py
```

You should see voltage, current, power, and battery percentage information.

```bash
Load Voltage:   3.652 V
Current:       -0.490 A
Power:          1.789 W
Percent:       54.3%
```
See: https://github.com/knowledgeescalation/OpenNannyApi/blob/main/sensors/ups-test.py.

## Audio System Configuration

{{< justify >}}
Setting up the audio system for OpenNanny is more complex than other components because it requires configuring both input (microphones) and output (speakers) over the I2S interface.
{{< /justify >}} 

### Microphone and Speaker Wiring


![wiring_mic_speakers](wiring_mic_speakers.png)
See: https://www.hackster.io/shiva-siddharth/make-your-own-google-voice-hat-9f96ca.

1. Connect the MEMS microphones (SPH0645LM4H) to the Raspberry Pi:
    - Connect VCC to 3.3V
    - Connect GND to ground
    - Connect BCLK to GPIO18
    - Connect LRCLK to GPIO19
    - Connect DATA to GPIO20
2. Connect the MAX98357A amplifier:
    - Connect VIN to 5V
    - Connect GND to ground
    - Connect BCLK to GPIO18 (shared with mic)
    - Connect LRCLK to GPIO19 (shared with mic)
    - Connect DIN to GPIO21
3. Connect your speakers to the amplifier output terminals ([parallel](https://www.granvozchina.com/info/how-to-wire-speakers-in-parallel-series-and-98026087.html)).

### EEPROM Configuration for Audio HAT Emulation

<img src="eeprom.png" alt="eeprom" style="width:30%; height:auto;"> 

To make both the microphones and speakers work together, we need to configure an EEPROM to emulate the [Google AIY audioHAT](https://aiyprojects.withgoogle.com/voice).

1. Connect the 24LC256 EEPROM to the Raspberry Pi:
    - Connect VCC to 3.3V
    - Connect GND to ground
    - Connect SDA to GPIO0 (SDA0)
    - Connect SCL to GPIO1 (SCL0)

2. Verify the EEPROM connection:

```bash
sudo i2cdetect -y 0
```

You should see a device at address `0x50`.

```bash
0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:                         -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
50: 50 -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --> EEPROM
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- -- 
```


3. Load the required kernel modules:

```bash
sudo modprobe i2c_dev
sudo modprobe at24
```

4. Create the device interface:

```bash
sudo su -
echo "24c256 0x50" > /sys/class/i2c-adapter/i2c-0/new_device
```

You should see your device.
```bash
ls /sys/class/i2c-adapter/i2c-0/
```

```bash
0-0050	delete_device  device  i2c-dev	mux_device  name  new_device  of_node  power  subsystem  uevent
```

```bash
ls /sys/class/i2c-adapter/i2c-0/0-0050/
```

```bash
0-00502  driver  eeprom  modalias  name  power	subsystem  supplier:regulator:regulator.0  uevent
```

5. You can play with [HAT tools](https://github.com/raspberrypi/hats) or simply use my blob (`voicehat.bin`). 

```bash
strings OpenNannyApi/voicehat.bin

R-Pi
Google, Inc.Google AIY audioHAT Board v1
```

```bash
xxd OpenNannyApi/voicehat.bin 

00000000: 522d 5069 0100 0200 7c00 0000 0100 0000  R-Pi....|.......
00000010: 4000 0000 5e69 9506 3673 da92 9142 be31  @...^i..6s...B.1
00000020: 52b7 df0d 0100 0100 0c1c 476f 6f67 6c65  R.........Google
00000030: 2c20 496e 632e 476f 6f67 6c65 2041 4959  , Inc.Google AIY
00000040: 2061 7564 696f 4841 5420 426f 6172 6420   audioHAT Board 
00000050: 7631 9290 0200 0100 2000 0000 0000 0000  v1...... .......
00000060: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000070: 0000 0000 0000 0000 0000 ed6e ffff ffff  ...........n....
```

Write the provided configuration data to the EEPROM:

```bash
sudo dd if=OpenNannyApi/voicehat.bin of=/sys/class/i2c-adapter/i2c-0/0-0050/eeprom
```

Verify.
```bash
dd if=/sys/class/i2c-adapter/i2c-0/0-0050/eeprom of=a.bin
```

6. Edit your boot configuration to enable the audio hardware:

```bash
sudo nano /boot/firmware/config.txt
```

Add these lines at the end:

```
dtoverlay=i2s-mmap
dtoverlay=googlevoicehat-soundcard
```

7. Configure your audio system:

```bash
sudo nano /etc/asound.conf
```

```bash
options snd_rpi_googlemihat_soundcard index=0

pcm.softvol {
    type softvol
    slave.pcm dmix
    control {
        name Master
        card 0
    }
}

pcm.micboost {
    type plug
    slave.pcm dsnoop
    
    }

pcm.!default {
    type asym
    playback.pcm "plug:softvol"
    capture.pcm "plug:micboost"
}

ctl.!default {
    type hw
    card 0
}
```

8. Reboot and test your audio setup:

```bash
sudo reboot
```

After rebooting, test recording and playback:

```bash
arecord -D plughw:0 -c2 -r 48000 -f S32_LE -t wav -V stereo -v test.wav
speaker-test -c2 --test=wav -w test.wav
```


```bash
## /boot/firmware/config.txt

# Uncomment some or all of these to enable the optional hardware interfaces
dtparam=i2c_arm=on
dtparam=i2s=on
dtparam=spi=on
dtparam=i2c_vc=on

# Automatically load overlays for detected cameras
camera_auto_detect=1

# Automatically load overlays for detected DSI displays
display_auto_detect=1

# Automatically load initramfs files, if found
auto_initramfs=1

# Enable DRM VC4 V3D driver
dtoverlay=vc4-kms-v3d
max_framebuffers=2

# Don't have the firmware create an initial video= setting in cmdline.txt.
# Use the kernel's default instead.
disable_fw_kms_setup=1

# Run in 64-bit mode
arm_64bit=1

# Disable compensation for displays with overscan
disable_overscan=1

# Run as fast as firmware / board allows
arm_boost=1

[cm4]
# Enable host mode on the 2711 built-in XHCI USB controller.
# This line should be removed if the legacy DWC2 controller is required
# (e.g. for USB device mode) or if USB support is not required.
otg_mode=1

[cm5]
dtoverlay=dwc2,dr_mode=host

[all]

dtoverlay=i2s-mmap
dtoverlay=googlevoicehat-soundcard
```


## Camera and Night Vision Setup

The camera system provides real-time video monitoring with night vision capabilities.

![camera](camera.png)

### Installing the Camera Module

1. Connect the CSI camera to the Raspberry Pi.
2. Enable the camera in Raspberry Pi configuration:

```bash
sudo raspi-config
```

Navigate to Interface Options → Camera → Enable.

Or see: /boot/firmware/config.txt.

3. Install required camera software:

```bash
sudo apt install -y python3-picamera2 libcamera-dev
```

4. Test the camera:

```bash
libcamera-still -q 80 -o test.jpg
```

5. Test camera with `opencv`.

Create virtual environment for python:
```bash
python -m venv --system-site-packages env
env/bin/activate
```

Install `opencv`.
```bash
pip install opencv-python
```

Create script and check if you can read a video frame.
```python
import cv2
from picamera2 import Picamera2

picam2 = Picamera2()
picam2.preview_configuration.main.size = (800,800)
picam2.preview_configuration.main.format = "RGB888"
picam2.preview_configuration.align()
picam2.configure("preview")
picam2.start()

im = picam2.capture_array()
print(im)
print()
print(type(im))
print(im.shape)
```

### IR LED Night Vision Control




![camera_back](camera2.png)

The standard camera module's IR LEDs consume significant power and emit visible red light. Our improved design allows software control of the IR lighting.

1. Modify the IR LED circuit:
    - Keep the positive terminal of the IR LEDs connected to 3.3V
    - Connect the negative terminal to the collector (C) of a 2N2222 NPN transistor
    - Connect the transistor emitter (E) to ground
    - Connect the transistor base (B) to GPIO pins through 1kΩ resistors
    - I used GPIO pins 6 and 26, but you can choose others
2. Test the IR LED control with the provided script:

```python
cd ~/OpenNannyApi/api
python
import led_control
led_control.right_on()
led_control.left_on()
led_control.right_off()
led_control.left_off()
```

<img src="npn.webp" alt="npn transistor" style="width:40%; height:auto;">




## WiFi Power Management Fix

{{< justify >}} 
By default, Raspberry Pi Zero's WiFi enters power-saving mode after periods of inactivity, which can disrupt the monitor's connection. To prevent this.
{{< /justify >}} 

1. Check your current power management status:

```bash
iwconfig
```

```bash
...
wlan0     IEEE 802.11  ESSID:"YOUR_SSID"  
          Mode:Managed  Frequency:2.412 GHz  Access Point: XX:XX:XX:XX:XX:XX   
          Bit Rate=57.7 Mb/s   Tx-Power=31 dBm   
          Retry short limit:7   RTS thr:off   Fragment thr:off
          Power Management:on
          Link Quality=52/70  Signal level=-58 dBm  
          Rx invalid nwid:0  Rx invalid crypt:0  Rx invalid frag:0
          Tx excessive retries:1  Invalid misc:0   Missed beacon:0
...
```

2. Disable WiFi power management permanently:

```bash
sudo nano /etc/rc.local
```

Add before the exit line:

```bash
/sbin/iwconfig wlan0 power off
```


3. Reboot & verify:
```bash
iwconfig
```

You should now see `Power Management:off` in the output.
```bash
...
wlan0     IEEE 802.11  ESSID:"YOUR_SSID"  
          Mode:Managed  Frequency:2.412 GHz  Access Point: XX:XX:XX:XX:XX:XX   
          Bit Rate=57.7 Mb/s   Tx-Power=31 dBm   
          Retry short limit:7   RTS thr:off   Fragment thr:off
          Power Management:off
          Link Quality=52/70  Signal level=-58 dBm  
          Rx invalid nwid:0  Rx invalid crypt:0  Rx invalid frag:0
          Tx excessive retries:1  Invalid misc:0   Missed beacon:0
...
```


## Setting Up the OpenNannyApi Software



This section describes setting up the software to run RESTful API for `OpenNanny`.

### Creating SSL Certificates for Secure Communication


For secure encrypted communication between the app and your OpenNanny.

1. Generate certificates using SimplePKI:

```bash
git clone https://github.com/knowledgeescalation/SimplePKI.git
cd SimplePKI
chmod +x *.sh
./gen-ca.sh
```

2. Follow the prompts to create your Certificate Authority:

- Country (two-letter country code)
- State/Province
- Locality/City
- Organization
- Common Name for your CA

The script will:

- Create necessary directories (root-ca, crl, certs, new_certs, csr)
- Generate an ECC private key for your CA
- Create a self-signed CA certificate valid for 5 years
- Generate a random serial number
- Create a `gen-cert.sh` script for issuing certificates

3. Generate a server certificate:

```bash
./gen-cert.sh
```

You'll be prompted to enter the server's DNS name. The script will:

- Generate an ECC key pair for the server
- Create a Certificate Signing Request (CSR)
- Set up proper DNS entries in the Subject Alternative Name extension
- Issue and sign the certificate with your CA

4. Copy the generated certificates to your OpenNannyApi directory.




### Configuring the InfluxDB Time-Series Database

InfluxDB stores environmental sensor data for historical analysis.

1. Create the InfluxDB configuration files:

```bash
cd ~/OpenNannyApi
mkdir influx
```

2. Create credential files for secure database access:

```bash
sudo nano /root/.env.influxdb2-admin-username
# Enter your admin username
sudo nano /root/.env.influxdb2-admin-password
# Enter your admin password
sudo nano /root/.env.influxdb2-admin-token
# Enter a secure token
```

3. Create the Docker Compose file as specified in the repository.

```bash
vim influx/compose.yaml
```

```yml
version: "3"

services:
  influxdb:
    image: influxdb:2
    container_name: influx
    ports:
      - 127.0.0.1:8086:8086
    environment:
      USER-UID: 1000
      USER_GID: 1000
      DOCKER_INFLUXDB_INIT_MODE: setup
      DOCKER_INFLUXDB_INIT_USERNAME_FILE: /run/secrets/influxdb2-admin-username
      DOCKER_INFLUXDB_INIT_PASSWORD_FILE: /run/secrets/influxdb2-admin-password
      DOCKER_INFLUXDB_INIT_ADMIN_TOKEN_FILE: /run/secrets/influxdb2-admin-token
      DOCKER_INFLUXDB_INIT_ORG: "INFLUX_ORG"
      DOCKER_INFLUXDB_INIT_BUCKET: "INFLUX_BUCKET"
    restart: always
    secrets:
      - influxdb2-admin-username
      - influxdb2-admin-password
      - influxdb2-admin-token
    volumes:
      - /home/"USER_NAME"/OpenNannyApi/influx/influx_data:/var/lib/influxdb2:rw
      - /home/"USER_NAME"/OpenNannyApi/influx/influx_config:/etc/influxdb2:rw
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
secrets:
  influxdb2-admin-username:
    file: ~/.env.influxdb2-admin-username
  influxdb2-admin-password:
    file: ~/.env.influxdb2-admin-password
  influxdb2-admin-token:
    file: ~/.env.influxdb2-admin-token
```

4. Set up InfluxDB as a system service:

```bash
sudo vim /etc/systemd/system/influx.service
```

```bash
[Unit]
Description=InfluxDB service with Docker Compose
PartOf=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=true
WorkingDirectory=/home/"USER_NAME"/OpenNannyApi/influx
ExecStart=/usr/bin/docker-compose up -d --remove-orphans
ExecStop=/usr/bin/docker-compose down

[Install]
WantedBy=multi-user.target
```

5. Enable and start influx service:
```bash
systemctl enable influx
systemctl start influx
```

6. Verify the database is running:

```bash
systemctl status influx
docker ps
ss -tulpn
```



If everything works try logging into the database via web browser (`127.0.0.1:8086`) over ssh.
```bash
ssh -L 8086:127.0.0.1:8086 \$USER_NAME@\$NANNY_IP
```

### Integrating Sensors with the Database

1. Configure the sensor integration:

```bash
vim ~/OpenNannyApi/sensors/.env
```

2. Add your InfluxDB credentials:

```
token=YOUR_INFLUX_TOKEN
influx_org=YOUR_ORG_NAME
influx_bucket=YOUR_BUCKET_NAME
```

3. Run `script.py`:
```bash
python sensors/script.py
```

4. If everything works create a system service for sensor data collection:

```bash
sudo vim /etc/systemd/system/sensors.service
```

```bash
[Unit]
Description=CO2 & UPS sensors
After=influx.service

[Service]
Restart=always
ExecStart=/home/"USER_NAME"/OpenNannyApi/sensors/env/bin/python /home/"USER_NAME"/OpenNannyApi/sensors/script.py
ExecStop=deactivate
[Install]
WantedBy=multi-user.target
```

5. Enable and start sensors service:

```bash
systemctl enable sensors
systemctl start sensors
```

6. After running for some time, you can create dashboards to visualize the environmental data:
```bash
ssh -L 8086:127.0.0.1:8086 \$USER_NAME@\$NANNY_IP
```

7. Create dashboards.

Co2 plot.
```python
from(bucket: "influx_bucket")
	|> range(start: v.timeRangeStart, stop: v.timeRangeStop)
	|> filter(fn: (r) => r["_measurement"] == "co2_sensor")
	|> filter(fn: (r) => r["_field"] == "co2")
```

Temperature plot.
```python
from(bucket: "influx_bucket")
	|> range(start: v.timeRangeStart, stop: v.timeRangeStop)
	|> filter(fn: (r) => r["_measurement"] == "co2_sensor")
	|> filter(fn: (r) => r["_field"] == "temperature")
```

Humidity plot.
```python
from(bucket: "influx_bucket")
	|> range(start: v.timeRangeStart, stop: v.timeRangeStop)
	|> filter(fn: (r) => r["_measurement"] == "co2_sensor")
	|> filter(fn: (r) => r["_field"] == "humidity")
```

Charge plot.
```python
from(bucket: "influx_bucket")
	|> range(start: v.timeRangeStart, stop: v.timeRangeStop)
	|> filter(fn: (r) => r["_measurement"] == "ups_sensor")
	|> filter(fn: (r) => r["_field"] == "charge")
```

Current plot.
```python
from(bucket: "influx_bucket")
	|> range(start: v.timeRangeStart, stop: v.timeRangeStop)
	|> filter(fn: (r) => r["_measurement"] == "ups_sensor")
	|> filter(fn: (r) => r["_field"] == "current")
```

8. Analyze the data.

![influx](influx.png)


### Setting Up the REST API Service

1. Generate a secure key for JWT authentication:

```bash
openssl rand -hex 32
```

2. Create a password hash:

```bash
htpasswd -bnBC 12 "" YOUR_PASSWORD
```

3. Configure the API environment:

```bash
vim ~/OpenNannyApi/api/.env
```

```python
SECRET_KEY=

USER_NAME=
FULL_USER_NAME=
USER_EMAIL=

USER_HASH=

MUSIC_PATH=DIRECTORY/TO/STORE/MP3/FILES
```

4. Debug API server:

- without certificate:
```bash
uvicorn main:app --host 0.0.0.0 --port 8888
```

- with certificate:
```bash
uvicorn main:app --host 0.0.0.0 --port 8888 --ssl-keyfile ./"KEY_NAME".key --ssl-certfile
```

5. If everything is working properly create a system service.

```bash
vim /etc/systemd/system/api.service
```

```bash
[Unit]
Description=API service
After=network.target

[Service]
WorkingDirectory=/home/"USER_NAME"/OpenNannyApi/api
ExecStart=/home/"USER_NAME"/OpenNannyApi/api/env/bin/uvicorn main:app --host 0.0.0.0 --port 8888 --ssl-keyfile ./"KEY_NAME".key --ssl-certfile ./"KEY_NAME".crt 
Restart=on-failure
PrivateTmp=true
ProtectSystem=full
User="USER_NAME"
Group="USER_NAME"

[Install]
WantedBy=multi-user.target
```

6. Enable and start api service:
```bash
systemctl enable api
systemctl start api
```

5. Test the API functionality (you can export `NANNY_IP` variable in your terminal).

- For TLS verification you can analyze the output of following command or use `wireshark`.

```bash
openssl s_client -connect $NANNY_IP:8888
```

```bash
---
No client certificate CA names sent
Peer signing digest: SHA384
Peer signature type: ECDSA
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 1202 bytes and written 395 bytes
Verification: OK
---
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Server public key is 384 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 0 (ok)
---
```

- Check API endpoint without token.
```bash
curl https://$NANNY_IP:8888/sensors
```

```bash
{"detail":"Not authenticated"}
```

- Check `token` endpoint with wrong username/password.

```bash
curl -X POST \
https://$NANNY_IP:8888/token \
-H 'accept: application/json' \
-H 'Content-Type: application/x-www-form-urlencoded' \
-d 'username=test&password=test'\
```

- Authenticate to `token` endpoint with your username/password and get JWT token.

```bash
curl -X POST \
https://$NANNY_IP:8888/token \
-H 'accept: application/json' \
-H 'Content-Type: application/x-www-form-urlencoded' \
-d 'username=YOUR_USERNAME&password=YOUR_SECRET'
```

```bash
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJkb21lY3pla2FwcCIsImV4cCI6MTc0MjU4NzE2NX0.kVRrPOtSrPM1qjXW_uv0RLnLFcytIDO45bXlB9gphM4","token_type":"bearer"}
```

- Check API endpoint without valid token.

```bash
curl https://$NANNY_IP:8888/sensors -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJkb21lY3pla2FwcCIsImV4cCI6MTc0MjU4NzE2NX0.kVRrPOtSrPM1qjXW_uv0RLnLFcytIDO45bXlB9gphM4'
```

You should receive sensor data in JSON format.
```bash
[{"name":"co2","value":"979"},{"name":"humidity","value":"42.14"},{"name":"temperature","value":"23.7"},{"name":"charge","value":"33.3"}]
```

- Use following script to test WebRTC without authentication.
```python
import asyncio
import websockets

async def webrtc_client():
    uri = "wss://NANNY_IP:8888/webrtc"  # Adjust host/port if needed
    
    async with websockets.connect(uri) as websocket:
        print("Connected to WebRTC server")

if __name__ == "__main__":
    asyncio.run(webrtc_client())
```

- Use following script to test WebRTC with authentication.
```python
import asyncio
import websockets
import json

async def webrtc_client():
    uri = "wss://NANNY_IP:8888/webrtc"  # Adjust host/port if needed
    token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJkb21lY3pla2FwcCIsImV4cCI6MTc0MjU4NzE2NX0.kVRrPOtSrPM1qjXW_uv0RLnLFcytIDO45bXlB9gphM4"  # Replace with a valid JWT token
    
    async with websockets.connect(uri, additional_headers={
        "Authorization": f"Bearer {token}"
    }) as websocket:
        print("Connected to WebRTC server")
        

if __name__ == "__main__":
    asyncio.run(webrtc_client())
```
 
 - To update python trust anchors run:
```bash
export REQUESTS_CA_BUNDLE=/path/to/your/certificate.pem
```

## Configuring the OpenNannyApp

[**OpenNannyApp**](https://github.com/knowledgeescalation/OpenNannyApp) is an Android application designed to connect with and control a [OpenNannyApi](https://github.com/knowledgeescalation/OpenNannyApi). The app provides a user-friendly interface to monitor environmental conditions, stream video feed, and play music through the connected monitoring device. In order to setup application do following steps.

1. Install the OpenNannyApp on your Android device.
2. Configure the app to connect to your OpenNanny:
```xml
# Edit app/src/main/res/values/strings.xml
<resources>
	<string name="app_name">OpenNannyApp</string>
	<string name="api_ip">NANNY_IP</string>
	<string name="api_user">USER_NAME</string>
	<string name="api_pass">USER_PASS</string>
</resources>
```
3. Add Root CA certificate (`app/src/main/res/raw/ca.crt`).
4. Navigate through the app to access different monitoring and control features.


## Security Considerations and Best Practices

When building a baby monitor, security should be a top priority:

- **Local Network Operation**: Unlike cloud-connected commercial monitors, OpenNanny operates entirely on your local network
- **Data Encryption**: All communication between the app and OpenNanny is encrypted using TLS
- **Authentication**: JWT tokens provide secure authentication for all API requests
- **No Cloud Storage**: Video feeds and sensor data never leave your network
- **Regular Updates**: Keep your Raspberry Pi OS and OpenNanny software updated
- **Change Default Credentials**: Always use strong, unique passwords for API access
- **Firewall**: Implement firewall rules using `iptables`, block all traffic except SSH and API.

## Troubleshooting Common Issues

- **Camera Not Detected**: Ensure the ribbon cable is properly seated and camera is enabled
- **Audio Issues**: Check I2S configuration and verify EEPROM data was written correctly
- **WiFi Disconnection**: Confirm power management is disabled and your router has a strong signal
- **Sensor Reading Errors**: Verify I2C connections and addresses with i2cdetect
- **Battery Drain**: Check current consumption with the UPS HAT monitoring tools

## Conclusion and Next Steps

{{< justify >}} 
The OpenNanny project we've explored represents a comprehensive approach with enterprise-grade security and expandability. By building your own smart baby monitor, you gain complete control over your data and can customize the system to your specific needs—all at a fraction of the cost of commercial alternatives with fewer features.

Whether you choose to implement the basic monitoring capabilities or expand into advanced AI features like computer vision with YOLO, voice capabilities, or local LLM integration, your DIY OpenNanny puts you in control of your baby's monitoring environment. The modular nature of the project allows you to start with core functionality and gradually enhance it as your skills and requirements evolve.

As the Raspberry Pi platform continues to improve with more powerful models and the open-source community contributes new innovations, the possibilities for enhancing your baby monitor will only expand. Your OpenNanny isn't just a useful tool for your family—it's also an opportunity to develop valuable skills in Linux, networking, sensors, and embedded systems programming that can be applied to many other projects.

We hope this guide inspires you to create a secure, feature-rich monitoring solution tailored to your family's needs. Happy building!
{{< /justify >}} 
