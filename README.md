# Owlet API

## Purpose

The purpose of this project is to reverse-engineer the Owlet Baby Monitor protocol to try to surface the data produced by the monitor to 3rd party applications (wall dashboard, analytics etc.)

## Progress

### 2016-08-04

First try to see how the protocol looks like. The current findings are:

* I routed all traffic using a proxy to my local machine
* The Owlet app uses API on following address https://ads-field.aylanetworks.com
* Here the app gets the credentials (looks like standard OAuth/OpenID refresh/access token combo)
* It also retrieves list of devices, their capabilities and triggers
* Part of the data is also a local LAN address of the base station
```
GET https://ads-field.aylanetworks.com/apiv1/devices.json HTTP/1.1
Host: ads-field.aylanetworks.com
Connection: keep-alive
Proxy-Connection: keep-alive
Accept: */*
User-Agent: Owlet/20160625 (iPad; iOS 9.3.2; Scale/2.00)
Accept-Language: sk-SK;q=1, en-SK;q=0.9
Authorization: auth_token hahaha
Accept-Encoding: gzip, deflate


HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Date: Thu, 04 Aug 2016 23:59:04 GMT
ETag: "1840ac0291aa8ce1842f3e214167d737"
Last-Modified: Wed, 14 Dec 2050 18:43:58 GMT
Server: Apache
Status: 200 OK
X-Frame-Options: sameorigin
X-Rack-Cache: miss
X-Request-Id: 90fd7b56-210a-4a76-b6fc-2f05acb2bbb4
X-Runtime: 0.1295
X-UA-Compatible: IE=Edge,chrome=1
Content-Length: 392
Connection: keep-alive

[{"device":{"product_name":"Owlet Baby Monitors","model":"AY001MTC1","dsn":"AC000W000258920","oem_model":"BabyOwl4","template_id":241,"mac":"60f189cc5210","lan_ip":"**172.17.5.184**","connected_at":"2016-08-04T23:11:41Z","key":200860,"lan_enabled":true,"has_properties":true,"product_class":null,"connection_status":"Online","lat":"52.5167","lng":"13.4","locality":"10409","device_type":"Wifi"}}]
```

* Over this HTTPS channel there is no realtime telemetry data transferred

After further digging I found that the app sends a request directly to the base to be registered (as I assume all the realtime telemetry info).

```
PUT http://172.17.5.184/local_reg.json HTTP/1.1
Host: 172.17.5.184
Content-Type: application/json; charset=utf-8
Content-Length: 77
Connection: keep-alive
Connection: keep-alive
Accept: */*
User-Agent: Owlet/20160625 (iPad; iOS 9.3.2; Scale/2.00)
Accept-Language: sk-SK;q=1, en-SK;q=0.9
Authorization: auth_token null
Accept-Encoding: gzip, deflate

{"local_reg":{"uri":"local_lan","notify":1,"ip":"**172.17.5.186**","port":40011}}

HTTP/1.1 202 Accepted

```

This tells the owlet to initiate a connection with a server at `ip` on the specified port with the specified uri (i.e. `http://172.17.5.186:40011/local_lan/...`. 

Running netcat gives request that comes from the owlet

```
$ nc -v -l 40011
Listening on [0.0.0.0] (family 0, port 40011)
Connection from [172.17.5.184] port 40011 [tcp/*] accepted (family 2, sport 50402)
POST local_lan/key_exchange.json HTTP/1.1
Host: 172.17.5.186
Accept: application/json
Content-Type: application/json
Content-Length: 106

{"key_exchange":{"ver":1,"random_1":"vHBzXO5HZ5/kBZlx","time_1":105605732147035,"proto":1,"key_id":49373}}
```

And running a simple flask server to respond with a `random_2` and `time_2` seems to keep the connection alive. 

```
from flask import Flask
from flask import jsonify

import time

app = Flask(__name__)

@app.route("/local_lan/key_exchange.json", methods=['GET', 'POST'])
def key_exchange_handler():

    d = {"random_2": "ABCDABCDABCDABCD", "time_2": long(time.time()) }
    
    return jsonify(d)

if __name__ == "__main__":
    app.run(debug=True, port=40011, host="0.0.0.0")
```

At this point there seems to be some type of get request required to prompt the device for the values. 

## Code

TBA
