+++
date = "2018-02-09T03:57:28+02:00"
draft = false
title = "Building Sensor Network: Jablotron Futura integration"

tags = [ "Personal DWH", "Jablotron", "Futura", "Modbus2MQTT"]
+++

### Jablotron Futura
Futura is a heat recovery unit, sort of heart of a house ventilation system and as such it can introduce complex information about the indoor air quality into my sensor network. But to be honest that fact wasn't the main driver for its integration.

### The driver
It was one issue I had with the unit and couldn't fix it within the standard product support. The behavior of the units depends on the settings you give it. The most important value is the required temperature.

I think there is a bit naive idea that if you set the optimal temperature, then the main requirement is to keep such temperature exactly and strictly on that level at any point of time. But that is infitely far from the reality. 

Functional heat recovery unit should think ahead a bit. So during a hot summer when the temperature during the night fall below the required level it cannot start recovering the heat to keep the exact required temperature, that would be definitely wasting the perfect opportunity to cool the house to cope better with the heat coming within the day.

And similarily for the cold winter. If we light our fireplace and the indoor temperature goes under the required level, I don't want the heat unit to switch to bypass mode immediately and waste all the heat.

Of course it is possible to always shift the required temperature in the mobile app, but honestly this is task for the machine not for any creative human being. 

Fortunately the solution is easy, just to set the required temperature way above the optimal in winter and way below the optimal in summer. Then everything works exactly how it should. Now we are geting close to the issue I had.

Unfortunately some of the settings get lost whenever the heat recovery unit  disconnects from the power supply. And the required temperature is one of them so it resets to the default value 21 degrees. Consulting this issue with product support they say the unit should keep all the setting but so far cannot find out why it is not so. 

### The idea
The good thing I learned from the support guy was that it is possible to communicate directly with the unit through Modbus TCP protocol. That new fact opened completely new direction in my thinking. There is no reason to have all the equipments in the house extra intelligent, it is perfectly enough if they can just collaborate with something smarter than they are.

### Futura integration
I already revealed the idea of sensor network acccess layers in the [first post](/building-sensor-network-bc/) of Building Sensor Network series.  

The main idea is to avoid the Babylon syndrom and unify the communication between all the devices in the sensor network.

We need to teach Futura to communicate over MQTT. And to keep thing simple, using silimar approach as for BigCLown modules. Introducing another translation gate, this time for Modbus protocol.

### Modbus to MQTT gate
I think we live in amazing times when the human knowledge is shared over the internet and it is just matter of minutes to gather enought information to solve most of the tasks we can face.

Sharing the source code moves us even further. Not surprisingly, I wasn't the first one having the idea of translation gate between Modbus and MQTT protokols. 

There was already [modbus2mqtt](https://github.com/owagner/modbus2mqtt) from Oliver Wagner. I tried to use it as it was but without much success, so I made [my own fork](https://github.com/lubomirkamensky/modbus2mqtt) and digged in a bit. There were just two compatibility issues related to latest Python and involved libraries.  The biggest task was to create [register definition](https://github.com/lubomirkamensky/modbus2mqtt/blob/master/futura.csv) file for the Futura.

I got Modbus register documentation for Futura from Jablotron support. And after some experiments started to understand how it all works.

The [register definition](https://github.com/lubomirkamensky/modbus2mqtt/blob/master/futura.csv) file defines what data available through Modbus we publish over MQTT.

For each attribute we need to specify following information: "Topic", "Register", "Size", "Format", "Frequency", "Slave", "FunctionCode"

Supporting our lazyness, it allowes to define defaults for constant values. 
For our purpose we define two big groups of attributes, one for Input registers, one for Holding registers.  Specification of each group starts with default definition specifying everything but the first two values.

Then for any  attribute we specify just the MQTT topic and position in Modbus Registers.
See the shortened version of the [register definition](https://github.com/lubomirkamensky/modbus2mqtt/blob/master/futura.csv) below. There is just one diference between the groups. The FunctionCode 4  for Input registers and FunctionCode 3 for Holding registers.

```
# Input Registers 
DEFAULT,,1,>H:,15,1,4
# 
temp_ambient,15
temp_fresh,16
temp_indoor,17
temp_waste,18
rh_ambient,19
# 
# Holding Registers 
DEFAULT,,1,>H:,15,1,3
# 
fventilation,0
boost_tm,1
circulation_tm,2
overpressure_tm,3
night_tm,4
```

Having the register definition file in place, we can test how it works

```
python3 modbus2mqtt.py --mqtt-topic futura --tcp HOST_IP --registers futura.csv
```

