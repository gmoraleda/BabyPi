<p align=center>
  <img width="365" height="141" src="logo.png">
</p>

## About
Baby monitor using a Raspberry Pi and the NoIR camera module.
This is intended as a summary of the steps that I followed to get a Raspberry Pi 3 working as a baby monitor. There are many guides out there, but for some reason or another, none of them was up to date and I could not get them working.

## Shopping list
- Raspberry Pi 3 (any model would do, but having WiFi is really convenient for this project)
- Power supply
- [NoIR camera module](https://shop.pimoroni.com/products/raspberry-pi-camera-module-v2-1-with-mount?variant=19833929799) (this opens the possibility of installing IR led/illuminator and be able to see at night)
- DHT22 / AM2302 temperature and humidity sensor (the 3-pin module version)
- SmartiPi case (Lego-compatible)
- Floating handle for action cameras (GoPro mount)

## Installation
### Basic setup
This whole project has been done with the Pi being managed via SSH. It is really convenient. Getting the Pi alive from scratch:

 1. Flash Raspberry Pi OS to the micro SD card with [Raspberry Pi Imager](https://www.raspberrypi.com/software/). In its settings, set the WiFi network, user and enable SSH, so the Pi joins the network on first boot.
 2. [Connect via SSH](https://www.raspberrypi.com/documentation/computers/remote-access.html#ssh) from a terminal or an SFTP client. On macOS I'm using [Cyberduck](https://cyberduck.io).

### Camera and DHT22
With the basic installation up and running we can shut down the Pi and start connecting the camera module. Once the camera module is [installed and tested](https://thepihut.com/blogs/raspberry-pi-tutorials/16021420-how-to-install-use-the-raspberry-pi-camera) (this is pretty straightforward), the next step is to connect the temperature sensor to the GPIO board. 

The version I bought is already prepared to be connected to the GPIO directly, but in case you have the version with 4 pins you need to install a resistor. For the 3-pin version, the wiring is done as follow:

    (+)   -> Pin 1
    (out) -> Pin 7
    (-)   -> Pin 6
From there on I followed the instructions [here](https://pimylifeup.com/raspberry-pi-humidity-sensor-dht22/).

The original `Adafruit_DHT` library is deprecated, so we use its CircuitPython replacement. Recent Raspberry Pi OS versions require a virtual environment for pip packages:

    sudo apt install libgpiod2
    python3 -m venv /home/pi/monitor/venv
    /home/pi/monitor/venv/bin/pip install adafruit-circuitpython-dht

The script that reads the temperature and humidity from the sensor is really simple:

    import time
    import adafruit_dht
    import board

    sensor = adafruit_dht.DHT22(board.D4)

    # DHT sensors often fail a single read, so retry a few times
    for _ in range(5):
        try:
            print(f"{sensor.temperature:0.1f}C {sensor.humidity:0.1f}%", end="")
            break
        except RuntimeError:
            time.sleep(2)
    else:
        print("Failed to retrieve data from humidity sensor", end="")

    sensor.exit()

_*Notice that even though the sensor is connected to the physical PIN 7, that PIN corresponds to GPIO4, therefore `board.D4`_

Store this in `/home/pi/monitor/humidity.py` and, if everything went as supposed, we should be able to query the sensor running the following command:

    /home/pi/monitor/venv/bin/python /home/pi/monitor/humidity.py

#### Schedule the reading 
With that in place we want to schedule a job using Cron to read the temperature every 5 minutes and store it in a log file. I created a second shell script to perform the readings and copy them to a log file and to a text file (*user_annotate.txt*) that will be shown in the camera live stream:


----------


##### temperature_script.sh

    #!/bin/bash

    log="/home/pi/monitor/logs/temperature.log"
    mkdir -p "$(dirname "$log")"

    # Read the sensor
    reading=$(/home/pi/monitor/venv/bin/python /home/pi/monitor/humidity.py)

    # Show it in the camera stream
    echo "$reading" > /dev/shm/mjpeg/user_annotate.txt

    # Output data to a log file
    echo "$(date +"%Y-%m-%d %T"): $reading" >> "$log"


----------


Make it executable (`chmod +x /home/pi/monitor/temperature_script.sh`), open the `pi` user's cron table (`crontab -e`, no `sudo` needed) and add the following line at the end:

    */5 * * * * /home/pi/monitor/temperature_script.sh

Make sure you write the right path there.

## RPi Cam Web Interface
[RPi Cam Web Interface](https://elinux.org/RPi-Cam-Web-Interface) allows us to connect to the camera using a browser. After following the installations instructions, we can access our camera typing the IP of our Raspberry Pi on any device connected to the same network. I'm using the standard configuration, using an Apache server.

Once the camera is running and visible there is only one missing step: adding the sensor information to the camera. There is an annotation prepared to display the content of `/dev/shm/mjpeg/user_annotate.txt` (therefore we copy it as part of the Cron script).

Under *Camera Settings -> Annotation* add `%a` to show the user annotation.

It is also convenient to start RPi Cam on every boot of the Raspberry Pi.

## Final thoughts
With everything connected and working this is how it looks:
![Browser Interface](https://imgur.com/ppIBUK5.jpg)

![Final Setup](https://i.imgur.com/gcjroVv.jpg)

The sensor needs to stay away from the Raspberry Pi case in order to provide accurate readings. The Raspberry Pi produces heat. Right now is hanging on the side. The Lego case opens many possibilities, so I could build a side box to hold the sensor.

The NoIR camera is able to record at night using IR lamps. The next step of this project could be installing a couple of IR led that automatically turn on at night using a light sensor or scheduling it after sunset (using some weather API).

There is no audio streaming. Although I bought a USB microphone, I haven't decided yet on how to attach the audio streaming to the camera feed. 
