---
title: Pico Power Monitor
date: 2022-12-29 12:00:00 0500
categories: [pico]
tags: [pico, meter]
---
# Pico Power Monitor

<div class="embed-container">
  <iframe
      src="https://www.youtube.com/embed/TWDl0wm8l34"
      width="700"
      height="480"
      frameborder="0"
      allowfullscreen="true">
  </iframe>
</div>

## Goal 
The goal is to be able to measure the power consumption of my house and send that information to [Home Assistant](https://www.home-assistant.io/) over MQTT where it will be logged.

![finalDesign](/assets/powerMonitor/topSidePicoPowerMonitor.jpg)

## Pre Requisite 
To get a good general understanding on how a project like this works i would read these articles
* [ac-power-intro](https://learn.openenergymonitor.org/electricity-monitoring/ac-power-theory/introduction)
* [optional video of similar project arduino](https://www.youtube.com/watch?v=TITtBkoaQ_s) 

## Equipment 
* raspberry pi pico W
* 2 TED 200A clamps
* 2 YHDC 100A SCT013 clamps
* 2 ADS1115 "External ADC"

## Project Description
To measure power for a given building, one of the easiest way to accomplish this is to use CT "Current Transform" clamps.  These wrap around the main wires in your house and as the name implies measure current.  The signal that comes out of from a CT is clamp is going to be oscillating signal between some positive current or voltage to the inverse of that.

## [CT Clamps](https://www.electroschematics.com/split-core-ct-primer/)

* [TED 200A Clamps model QX-201-CT](https://web.archive.org/web/20111021055834/http://jarv.org/2009/07/home-power-monitoring/)
These clamps are nice because they can handle a high current and are fairly sturdy clamps.  They also have a burden resistor inside of them which limits the peak to peak voltage to +-3V

* YMDC SCT013-000
These clamps fairly cheap all around but they get the job done.  There 100A and they don't include a burden resistor.  So you'll have to supply one to give the clamp a given voltage range.  There are lot of [great example](https://olimex.wordpress.com/2015/09/29/energy-monitoring-with-arduino-and-current-clamp-sensor/) on using these clamps.  Also [openEnergyMonitor](https://learn.openenergymonitor.org/electricity-monitoring/ct-sensors/yhdc-sct-013-000-ct-sensor-report) gave a extensive review of the product 

## ADC - **ADS1115**

Is a external ADC that can communicate over I2C.  This ADC has 4 inputs but there muxed, so at any given time you can only one.  But for this given example this adc has some benefits over most adc's built into microcontrollers.

### Upsides
* can take differential inputs 
* 16 bit sensitivity  
* relatively cheap
* very popular

### Downsides
* relatively slow (max 860 samples per second "SPS")

## Layout
So when it comes to wiring up the board the most important thing comes down to making sure you have a burden resistor across the clamps that need it (so any clamp thats rated in amps).  The next thing to consider is that the ADC can only handle +.3 of the input vcc that you give it.  So if you feed it 3.3 in theory you could only input a voltage of 3.6 to it without damaging it.  So even if you set the ADC to be +-4.096 if the vcc is set to 3.3 you won't be able to get the full range.  Lastly the ADS1115 can have 4 different i2c address based on the "ADDR","ALRT" pins.  So if you need multiple ADS1115 you'll need to properly set them.

![layout of the pico](/assets/powerMonitor/boardLayoutPicoElectricityMonitor.png)

![layout of the pico](/assets/powerMonitor/bottomSidePicoPowerMonitor.jpg)

## [Software for the Pico](https://github.com/brendena/pico_power_monitor)
Software can be found [here](https://github.com/brendena/pico_power_monitor).  On the repo it should describe to you how to actually setup the project.


## Home Assistant

### Setting up

To get Home Assistant up and running i choice to add it to a [virtual machine](https://www.home-assistant.io/installation/windows/), since it was the easiest way to get it working on my pc while still having the ability to [have add on's](https://www.home-assistant.io/installation/#compare-installation-methods).  If you install Home Assistant with docker you can't install add-ons.  

![Ways to install Home Assistant](/assets/powerMonitor/wayToInstallHomeAssistant.png)

## Home Assistant running Virtual Box
![virtual box Home Assistant](/assets/powerMonitor/homeAssistantVirtualBox.png)


### Configuring sensor
To log the values from our pico to your Home Assistant.  We need to have a MQTT broker enabled and then [link the sensors data to Home Assistant](https://community.home-assistant.io/t/energy-measurement-with-mqtt-sensor/424558).  The way to do this is by editing the config files and adding a mqtt sensor and a template sensor. 



```YAML
mqtt:
    sensor:
    -   name: "mainPowerLeft"
        state_topic: "mainPower/leftMain"
        unique_id: "leftMain"
        device_class: "energy"
        unit_of_measurement: "kWh"
        value_template: "{{ value | float | round (5) }}"
        state_class: "total_increasing"
        last_reset_topic: 'fake/last_reset'
        last_reset_value_template: "1970-01-01T00:00:00+00:00"
        

template:
    sensor:
    -   name: "mainPowerLeft"
        unit_of_measurement: "kWh"
        device_class: energy
        state_class: total_increasing
        last_reset: '1970-01-01T00:00:00+00:00'        
```

Important fields
* device_class: "energy"
* unit_of_measurement: "kWh" 
* state_class: "total_increasing" - debated online some say "monitor"
* last_reset_value_template: "1970-01-01T00:00:00+00:00"

These are what allow you to bind with the energy dashboard.  If its not setup properly this sensor will not show up as a option.

Once you have this you can [add your power to the "energy"](https://youtu.be/W7mVagcssZY?t=253) dashboard.  By going to \[setting\] -> \[Dashboards\] -> \[energy\] -> \[Add Consumption\]

Then after a few hours you'll start to get a graph like this

![Home Assistant energy consumed](/assets/powerMonitor/energyConsumed.png)


From here you should be all good.