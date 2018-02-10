+++
date = "2018-02-10T03:57:28+02:00"
draft = false
title = "Building Sensor Network: Neurio integration"

tags = [ "Personal DWH", "Neurio", "JSON2MQTT"]
+++

### Neurio
[Neurio Home Energy Monitor](https://www.neur.io/energy-monitor/) is one of my smart IoT sensors which offers nice cloud app. This one monitors the detailed house power consumption.

When I started to build my own sensor network independent of the cloud availability, I was pleased to find out that, not like Netatmo sensors, Neurio offers also local access to the actual measurements. Then the task was clear, I need to have power consumption information available through MQTT.

### Neurio local access
The Neurio Sensor implements a JSON format sensor value output. By calling http://IP_address/current-sample the sensor returns latest measurement value.

So in my case http://192.168.88.189/current-sample returns now:

```
{"sensorId":"0x0000C47F510180C9","timestamp":"2018-02-10T09:44:09Z","channels":[{"type":"PHASE_A_CONSUMPTION","ch":1,"eImp_Ws":4979583905,"eExp_Ws":1792905,"p_W":98,"q_VAR":-45,"v_V":233.666},{"type":"PHASE_B_CONSUMPTION","ch":2,"eImp_Ws":4845889558,"eExp_Ws":3195780,"p_W":789,"q_VAR":-214,"v_V":232.286},{"type":"PHASE_C_CONSUMPTION","ch":3,"eImp_Ws":6931117926,"eExp_Ws":15110144,"p_W":112,"q_VAR":-86,"v_V":235.431},{"type":"CONSUMPTION","ch":4,"eImp_Ws":16750543658,"eExp_Ws":14269510,"p_W":999,"q_VAR":-344,"v_V":233.794}],"cts":[{"ct":1,"p_W":98,"q_VAR":-45,"v_V":233.666},{"ct":2,"p_W":789,"q_VAR":-214,"v_V":232.286},{"ct":3,"p_W":112,"q_VAR":-86,"v_V":235.431},{"ct":4,"p_W":0,"q_VAR":0,"v_V":233.623}]}
```

### Time for Jason to MQTT translation gate
This time the short Google research didn't return any existing app and to be honest, I was happy for that. It is refreshing to go on my own in the simplest possible way from time to time.

The idea was simple, to create JSON2MQTT translation gate which accept as parameter URL of a Json resource and some template specifying how to cope with Json data.

Python is very powerful in simplicity of JSON like data transformation and it was very logical to utilize the power of list/set comprehension for the transformation template or mapping file if you will.

The simple template can look like this:

```
[ (i['type'],i['p_W']) for i in dataSet['channels'] ]
```

where "dataSet" refers to the root in JSON dictionary tree. Basically, it creates list of channel types and power in watts values.

And this list is then easily published over MQTT using Eclipse Paho for Python library, resulting in something like:

```
neurio/PHASE_A_CONSUMPTION 116
neurio/PHASE_B_CONSUMPTION 80
neurio/PHASE_C_CONSUMPTION 143
neurio/CONSUMPTION 338
```

See the complete [source code](https://github.com/lubomirkamensky/json2mqtt)

The final step is as always, adding the new gate into pm2 processes

```
pm2 start /usr/bin/python3 --name "json2mqtt-neurio" -- /home/luba/Git/json2mqtt/json2mqtt.py --json http://192.168.88.189/current-sample --map neurio.txt --mqtt-topic neurio
pm2 save
```

