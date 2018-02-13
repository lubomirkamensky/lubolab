+++
date = "2018-02-12T03:57:28+02:00"
draft = false
title = "Building Sensor Network: SMS gateway, Node-RED and smart alerts"

tags = [ "Personal DWH", "SMS-Gate", "Node-RED"]
+++

### SMS gateway
Before designing some simple flows in Node-RED, lets's create functional SMS gateway. The critical communication channel in the sensor network I build and describe in my blog. Having SMS gateway, we can create first practical use cases utilizing the data in access layers. Although there is finished just the first access layer now, the MQTT. 

I bought a cheap 3g USB dongle [Huawei E3531](https://consumer.huawei.com/en/mobile-broadband/e3531/specs/) and inspired by article [Sending SMS on a Raspberry Pi](https://escapologybb.com/send-sms-from-raspberry-pi/) started the installation.

First this:
```
sudo apt-get install ppp usb-modeswitch usb-modeswitch-data
```

Then some configuration:
```
sudo nano /etc/udev/rules.d/40-usb_modeswitch_custom.rules
```

adding this content:
```
# Huawei E3131h-2
ATTRS{idVendor}=="12d1", ATTRS{idProduct}=="15ca", RUN+="usb_modeswitch '%b:%k'"
```

And another configuration:
```
sudo nano /etc/usb_modeswitch.d/12d1\:15ca
```

adding this content:
```
#Huawei E3131 (Variant)
TargetVendor= 0x12d1
TargetProductList="1506"
MessageContent="55534243123456780000000000000011062000000100000000000000000000"
```

Then reboot with the connected dongle and check whether it is mounted.

![Desktop](/images/2018/02/3gdongle1.png)

This setup worked for me, it is a compilation of many sources. This particular dongle [Huawei E3531](https://consumer.huawei.com/en/mobile-broadband/e3531/specs/) is from [Czech market](https://modemy.heureka.cz/huawei-e3531/) and it was manufactured for O2 operator. I tested the setup for both Raspberry-pi and my tablet with Lubuntu getting the same output.

### Installing Gammu
To send SMS we need to install Gammu. 
```
sudo apt-get install gammu
```

and make some configuration:
```
sudo gammu-config
```

Within the configuration, I only changed the port according to the screen above (The first one from the three listed).
```
Port: /dev/ttyUSB1
Connection: at19200
Model: empty
Synchronize time: yes
Log file: leave empty
Log format: nothing
Use locking: leave empty
Gammu localisation: leave empty
```

Then tested sending an SMS to number 7XXX1XXX1
```
echo "test" | sudo gammu sendsms TEXT 7XXX1XXX1
```

It was a success, so I started another installation and setup to create simple file based interface for sending SMS. The idea is that new SMS is created as a text file in SMS outbox directory. Then it is very easy to integrate with Node-RED or anything else.

```
sudo apt-get install gammu-smsd
sudo nano /etc/gammu-smsdrc
sudo service gammu-smsd start
```

Then making sure the service is running
```
service --status-all | grep gammu
```

And to make sure it works with any pm2 process running under my user
```
sudo chown -R luba:luba /var/spool/gammu/outbox
```

Finally the SMS test, file name is "OUT" + phone number
```
echo "Hello Earth, Mars calling" > /var/spool/gammu/outbox/OUT7XXX1XXX1.txt
```

And success again. You can also change some setting in the config. Like shortening the timeouts. I put commtimeout = 1 and sendtimeout = 20.
```
sudo nano /etc/gammu-smsdrc
```


### Smart alerts
Finally, we are here. Something really practical. 

To send SMS alerts I use one of the Czech mobile virtual operators which offer free calls and SMS within its network. Having the cheapest tariff, the total monthly costs for one SIM is around 0.15 Eur. 

Surprisingly, I didn't find a perfect app for setting up smart alarms on android device. I would expect many of them, offering to select the communication channel, like SMS, email, Twitter, Telegram and so on. Then making alarm definition using filters based on a combination of senders, text and more, and finally defining the properties of alarm, sound, light, vibration effects. Okay, so far just dreams.

In reality, I have found just one app which was usable for the purpose but still, with limitations. It was [SMS Alarms](https://play.google.com/store/apps/details?id=com.unt.usms&hl=cs). It is nice and functional, but it can only define alarm based on the SMS sender number. Adding filters on text keywords, it would be perfect. 

#### Node-RED Doorbell SMS alarm
After building the smart Doorbell in the post [IoT Doorbell: with BigClown](/iot-doorbell-bc/), it is the time to see it in action. I take for granted that we have the setup described in the post [Building Sensor Network: with BigClown](/building-sensor-network-bc/) which includes Node-RED running as a pm2 process. 

Let's  create simple Node-RED flow now. Starting with "mqtt in" node:
```
Server: localhost:1883
Topic: node/carpinus/push-button/-/event-count
Qos:2
```

Then "function" node:
```
Name: DOORBELL RINGING
Function: 
if (msg.payload>0) {
msg.payload = "DOORBELL RINGING"
return msg;
}
Output:1
```

And finally the last, "file" node:
```
Filename: /var/spool/gammu/outbox/OUT7XXX1XXX1.txt
Action: append to file
Add newline (\n) to each payload? Yes
```

After deploying, the whole flow looks like this
![Desktop](/images/2018/02/flow1.png)

And here is the screen video of my phone. I started the stopwatch, pressed the doorbell and ...

<iframe width="560" height="315" src="https://www.youtube.com/embed/5yyMZo36UJE" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>

Of course I would like to have a different kind of alarm for a different type of event, unfortunately, considering the available app, this can be done only if the sender number varies. The alerts coming from my SMS gateway differs only in the text message, so it is important to define clear message and select the alarm alarming not too much nor too little

#### Node-RED Futura SMS alarm
The next flow  is very similar, this time it takes care about the most important alert, coming from the heat recovery unit, integrated to my sensor network in [this post](/building-sensor-network-futura/).

Again starts with "mqtt in" node:
```
Server: localhost:1883
Topic: futura/temp_set
Qos:2
```

Then "function" node:
```
Name: FUTURA TEMPERATURE RESET
Function: 
if (msg.payload==210) {
msg.payload = "FUTURA TEMPERATURE RESET"
return msg;
}
Output:1
```

And as the "file" node stays the one from prior flow

#### Node-RED Power Outage SMS alarm
This alarm is based on MQTT messages introduced withing the [prior post](/building-sensor-network-battery/) and again it follows the same pattern as the two flows above, starting with "mqtt in" node:
```
Server: localhost:1883
Topic: lubuntu/battery/state
Qos:2
```

Then "function" node:
```
Name: ELECTRICITY OUTAGE
Function: 
if (msg.payload=="discharging") {
msg.payload = "ELECTRICITY OUTAGE"
return msg;
}
Output:1
```

And the "file" node again the same. The whole picture including all three flows is here:
![Desktop](/images/2018/02/flow2.png)