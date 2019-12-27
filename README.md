<p align=center>
  <img width="365" height="141" src="logo.png">
</p>

## About
Baby monitor using a Raspberry Pi and the NoIR camera module.
This is inteded as a summary of the steps that I followed to get a Raspberry Pi 3 working as a baby monitor. There are many guides out there, but for some reason or another, none of them was up to date and I could not get them working.

## Hardware
- Raspberry Pi 3 (any model would do, but having wifi is really convinient for this project)
- Power supply
- [NoIR camera module](https://shop.pimoroni.com/products/raspberry-pi-camera-module-v2-1-with-mount?variant=19833929799) (this opens the possibility of installing IR leds/illuminator and be able to see at night)
- [DHT22 temperature and humidity sensor](https://www.ebay.de/itm/AM2302-DHT22-Digital-Temperatur-und-Feuchtigkeits-Sensor-Modul-SE08004/271574585686)
- [SmartiPi case](https://shop.pimoroni.com/products/smartipi-case-for-raspberry-pi)
- [Floating handle for action camera](https://www.ebay.de/itm/Water-Floating-Diving-Buoyancy-Selfie-Stick-Handle-Accessories-For-Gopro-gib/184044745464)

## Installation
### Ground installation
This whole project has been done with the Pi being managed via SSH. It is really convenient. There are four main steps to get the Pi alive from stratch:

 1. [Install](https://www.raspberrypi.org/documentation/installation/installing-images/) Raspbian in the micro SD card.
 2. [Create a wpa_supplicant.conf file](https://www.raspberrypi-spy.co.uk/2017/04/manually-setting-up-pi-wifi-using-wpa_supplicant-conf/) to connect the Pi to our wifi network.
 3. [Enable SSH](https://www.raspberrypi.org/documentation/remote-access/ssh/) to access the Pi remotely via terminal or with an SFTP client. On macOS I'm using [Cyberduck](https://cyberduck.io).

### Camera and DHT22
With the basic installation up and running we can shut down the Pi and start connecting the camera module. Once the camera module is [installed and tested](https://thepihut.com/blogs/raspberry-pi-tutorials/16021420-how-to-install-use-the-raspberry-pi-camera) (this is pretty straightforward), the next step is to connect the temperature sensor to the GPIO board. 

The version I bought is already prepared to be conneected to the GPIO directly, but in case you have the version with 4 pins you need to install a resistor. For the 3-pin version, the wiring is done as follow:

    (+)   -> Pin 1
    (out) -> Pin 7
    (-)   -> Pin 6
From there on I followed the instructions [here](https://pimylifeup.com/raspberry-pi-humidity-sensor-dht22/).

The python script (we need to install python and the Adafruit_DHT library) used to read the temperature and humidity from the sensor is really simple:

    import Adafruit_DHT

    DHT_SENSOR = Adafruit_DHT.DHT22
    DHT_PIN = 4

    humidity, temperature = Adafruit_DHT.read_retry(DHT_SENSOR, DHT_PIN)
    
    if humidity is not None and temperature is not None:
        print("{1:0.1f}C {0:0.1f}%".format(humidity, temperature))
    else:
        print("Failed to retrieve data from humidity sensor")

Store this in a file called `humidity.py` and, if everything went as supposed, we should be able to query the sensor running the following command:

    python3 ~/humidity.py

#### Schedule the reading 
With that in place we want to schedule a job using [cron](https://www.raspberrypi.org/documentation/linux/usage/cron.md) to read the temperature every 5 minutes and store it in a log file. I created a second shell script to perform the readings and copy them to a log file and also to a text file (*user_annotate.txt*) that will be shown in the livestream of the cam:


----------


##### temperature_script.sh

    #! /bin/bash
    
    log="/home/pi/monitor/logs/"
    
    #Run the client
    python3 /home/pi/monitor/temperature.py > temperature.txt
    cp temperature.txt /dev/shm/mjpeg/user_annotate.txt
   
    # Output data to a log file
    TEMPERATURE=`echo "$OUTPUT"`    
    echo "$(date +"%Y-%m-%d %T" ): ""$TEMPERATURE" >>"$log"temperature.log


----------


We need to open the cron table (`sudo crontab -e`), and add the following line at the end:

    */5 * * * * /home/pi/monitor/temperature_script.sh
