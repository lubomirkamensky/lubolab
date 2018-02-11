+++
date = "2018-02-11T03:57:28+02:00"
draft = false
title = "Building Sensor Network: Computer Battery Monitor"

tags = [ "Personal DWH", "Battery-Monitor2MQTT"]
+++

### Battery Monitor over MQTT
It can sound quite obscure to publish Computer Battery status over MQTT. But I think it is on the contrary, absolutely essential thing. Let me explain.

In fact, I was about to write a post about smart alerts, to move finally to something really practical after so many posts focusing just on building the sensor playground. Honestly, I postponed such topic already several times. Always something popped up, what was absolutely essential to cover before moving further. And I promise this is the last time.

As I described in prior posts, the heart of my sensor network is a Linux tablet receiving data from all the sensors. Disconnected from power supply the tablet can continue processing sensor data several hours on the batteries.  

<div class="c_alert c_alert-success"><i class="fa fa-exclamation-triangle"></i> 
This is extremely important for security sensors as every experienced burglar disconnects the power supply first. But the continuity of measurement has always its value. 
</div>

In case of power outage, the local network and internet access is probably out of function so it is good to have 3g usb dongle directly connected to the tablet and use it as SMS gateway for critical alerts.

<div class="c_alert c_alert-success"><i class="fa fa-info-circle"></i>
I consider the event of disconnecting from the power supply to be one of the most important events in our sensor network. And that's the motivation to add the tablet battery monitor into our first access layer, MQTT.
</div>

It is definitely useful to know about every power outage in our house immediately and then also to monitor the outage duration. Also, it is useful to be informed how long the tablet is going to last on the batteries.

### The solution
It is going to be just another gateway to the MQTT access layer. The approach is similar as for the prior one, [json2mqtt](https://github.com/lubomirkamensky/json2mqtt). Just instead of sourcing some external file, this time we have to gather the required information directly from our OS.

Okay, so the publishing part is fixed, we just need to learn how to gather the battery status information from a Linux OS. No reason to reinvent the wheel, such project is already here [battery-monitor](https://github.com/maateen/battery-monitor). Thanks for sharing [Maateen](https://github.com/maateen)!

So I simplified and customized bit the core part of it, the class BatteryMonitor
```
class BatteryMonitor:
    raw_battery_info = ''
    processed_battery_info = {}
    remainingFlag = True

    def __init__(self):
        self.raw_battery_info = self.get_raw_battery_info()
        self.get_processed_battery_info()
        self.on_ac_power = subprocess.call(['on_ac_power'])

    def get_raw_battery_info(self):
        command = "acpi -b"
        raw_info = subprocess.check_output(command,
                                           stderr=subprocess.PIPE,
                                           shell=True)
        return raw_info

    def get_processed_battery_info(self):
        in_list = (self.raw_battery_info.decode("utf-8", "strict").lower().strip('\n')
                   .split(": ", 1)[1].split(", "))

        self.processed_battery_info["state"] = in_list[0]
        self.processed_battery_info["percentage"] = in_list[1]
        try:
            self.processed_battery_info["remaining"] = in_list[2]
        except Exception as exc:
            self.remainingFlag = False

        return self.processed_battery_info
```

and placed it in the new tool [battery-monitor2mqtt](https://github.com/lubomirkamensky/battery-monitor2mqtt).

The usage is decribed in the [readme file](https://github.com/lubomirkamensky/battery-monitor2mqtt/blob/master/README.md). 

Then I tested it on my chromebook, starting on power supply and then disconnected:
![Chromebook](/images/2018/02/battery_chrome.png)

the tablet, starting on power supply and then disconnected:
![Tablet](/images/2018/02/battery_lubuntu.png)

and finally, even it doesn't have much sense, on my desktop computer without batteries. Here just on power supply, I guess everybody can imagine the dead screen: 
![Desktop](/images/2018/02/battery_ubuntu.png)

Each environment surprised with something new, so the tool can hopefully cope with most of the hardware now.

After successful testing is the new tool just another pm2 process on the tablet.
```
pm2 start /usr/bin/python3 --name "battery-monitor2mqtt" -- /home/luba/Git/battery-monitor2mqtt/battery-monitor2mqtt.py --mqtt-topic lubuntu
pm2 save
```

The collection is growing. Here is the complete list now:
![Desktop](/images/2018/02/pm2list2.png)
