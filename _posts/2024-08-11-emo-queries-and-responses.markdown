---
layout: post
title:  "EMO's smarts are in the cloud!"
author: "0xd34d"
date:   2024-08-11 14:00:00 -0800
categories: emo hacking
tags: emo hacking
---

# An exploration into how EMO handles user queries

I'm always curious how things work.  Mechanically, electrically or digitally I don't discriminate.  Having owned EMO for about 8 months I figured it was time to start to pick at what makes EMO tick.  This post will focus on one such endeavor in tryin to understand how EMO handles responding when you say his name and he replies with "What?" waiting for your query.

## Where to begin?
Without access to EMO's source code or a decrypted firmware dump to peruse he's essentially a black box, albiet with legs and wearing a pair headphones, so we're going to need figure out a starting point.  While I've disassembled EMO in the past, I wanted to start with a method that didn't require opening EMO up and probing around, that's probably a project to tackle in the future :grin:.

Two possibilities came to mind.  EMO has both bluetooth and WiFi capabilities so maybe there is something I could exploit with one of these?  Bluetooth is cool and all but based on the functionality provided by the mobile companion app I didn't think there was anything too interesting there.  I've worked on reversing EMOs BLE communications and will write about that in a future post :D.  Anyhow, that leaves us with WiFi.

### sniffing them packets
The obvious choice to accomplish capturing packets from EMO was to use my laptop's WiFi adapter as a hotspot that would share internet via the ethernet port.  Once the WiFi hostpot was setup I used the EMO app on Android to connect EMO to the hotspot.

With everything setup it was time to power down EMO, fire up Wireshark, and power EMO back up.  In less than a minute traffic starting filtering through Wireshark.  Woo-Hoo!  Hold up, let's not get too excited yet.  Look at those packets :bowtie:
![screenshot of wireshark showing captured packets]({{ site.baseurl }}/assets/wireshark_capture_emo_packets.png)
The bad news is the packets are encrypted using TLSv1.2, the good news is the packets are encrypted using TLSv1.2.

<br>
### bringing in the middle man
![screenshot of mitmproxy showing captured packets]({{ site.baseurl }}/assets/inspecting_packets_with_mitmproxy.png)

Request:
```
GET /time?tz=America/Los_Angeles HTTP/1.1
Host: api.living.ai
Connection: close
Secret: 
content-length: 0
```
Response:
```
HTTP/1.1 200 OK
Server: nginx
Date: Fri, 27 Jan 2023 17:02:13 GMT
Content-Type: application/json; charset=utf-8
Transfer-Encoding: chunked
Connection: close
Strict-Transport-Security: max-age=31536000

23
{"time":1674838933,"offset":-28800}
0
```

Request:
```
GET /token/b8d61aaaf162 HTTP/1.1
Host: api.living.ai
Connection: close
Secret: igZ_haD7wuHeMszUE8RBNQ
content-length: 0
```
Response:
```
HTTP/1.1 200 OK
Server: nginx
Date: Fri, 27 Jan 2023 17:02:15 GMT
Content-Type: application/json; charset=utf-8
Transfer-Encoding: chunked
Connection: close
Strict-Transport-Security: max-age=31536000

e7
{"access_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2NzQ4NDI1MzUsInN1YiI6ImI4ZDYxYWFhZjE2MiIsIm5iZiI6MTY3NDgzODkzNSwiaWF0IjoxNjc0ODM4OTM1fQ.bbYP4yevUwQ2S5e6XTaWkSq327rcKMNy0QH5w9NHm1c","expire_in":3600,"type":"Bearer"}
0
```

Request:
```
POST /emo/report/info HTTP/1.1
Host: api.living.ai:443
secret: o40ExzaWPNTu88_QZv6Q9Q
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2NzQ4NDI1MzUsInN1YiI6ImI4ZDYxYWFhZjE2MiIsIm5iZiI6MTY3NDgzODkzNSwiaWF0IjoxNjc0ODM4OTM1fQ.bbYP4yevUwQ2S5e6XTaWkSq327rcKMNy0QH5w9NHm1c
Content-Type: application/json
Connection: close
Content-Length: 187

{"ver":24,"city":"Seattle","tzone":"America/Los_Angeles","lang":"","age":222,"wkup":41039,"kcrash":"0/0.00","batt":4246,"drop":0,"beh":0,"touch":[1100,932,946],"wifi":-39,"mic":[0,0,0,0]}
```
Response:
```
HTTP/1.1 200 OK
Server: nginx
Date: Fri, 27 Jan 2023 17:02:18 GMT
Content-Type: application/json; charset=utf-8
Transfer-Encoding: chunked
Connection: close
Strict-Transport-Security: max-age=31536000

f
{"result":"OK"}
0
```
Request:
```
GET /emo/ota/version HTTP/1.1
Host: api.living.ai
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2NzQ4NDI1MzUsInN1YiI6ImI4ZDYxYWFhZjE2MiIsIm5iZiI6MTY3NDgzODkzNSwiaWF0IjoxNjc0ODM4OTM1fQ.bbYP4yevUwQ2S5e6XTaWkSq327rcKMNy0QH5w9NHm1c
Secret: QQa7LqSpu06Vmdh281JV3A
Content-Type: application/x-www-form-urlencoded
Connection: Keep-Alive
Keep-Alive: timeout=300, max=1000
content-length: 0
```
Response:
```
HTTP/1.1 200 OK
Server: nginx
Date: Fri, 27 Jan 2023 17:02:22 GMT
Content-Type: application/json; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
Strict-Transport-Security: max-age=31536000

29
{"version-name":"1.7.0","version-num":24}
0
```
Request:
```
GET /emo/notice/latest HTTP/1.1
Host: api.living.ai
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2NzQ4NDI1MzUsInN1YiI6ImI4ZDYxYWFhZjE2MiIsIm5iZiI6MTY3NDgzODkzNSwiaWF0IjoxNjc0ODM4OTM1fQ.bbYP4yevUwQ2S5e6XTaWkSq327rcKMNy0QH5w9NHm1c
Secret: QQa7LqSpu06Vmdh281JV3A
Content-Type: application/x-www-form-urlencoded
Connection: Keep-Alive
Keep-Alive: timeout=300, max=1000
content-length: 0
```
Response:
```
HTTP/1.1 200 OK
Server: nginx
Date: Fri, 27 Jan 2023 17:02:22 GMT
Content-Type: application/json; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
Strict-Transport-Security: max-age=31536000

28b
{"code":200,"errmessage":"ok","notice":{"0":{"id":6,"message":"do you know deep breathing is one of our easiest tools to combat issues like stress and anxiety, reduce, pain, high blood pressure and even aide in digestion. so, remember to take a deep breath."},"1":{"id":7,"message":"besides playing with me, being close to nature is also a way to keep you in a good, mood."},"2":{"id":8,"message":"don't regret anything that ever made you smile."},"3":{"id":9,"message":"appreciate the moments when you are in them. in a blink, they are but memories."},"4":{"id":10,"message":"accept what, is. let go of what, was, and have faith, in what will be."}}}
0
```
Request:
```
GET /emo/weather/forecast?city=Seattle&later=0&lon=0.00000&lat=0.00000 HTTP/1.1
Host: api.living.ai
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2NzQ4NDI1MzUsInN1YiI6ImI4ZDYxYWFhZjE2MiIsIm5iZiI6MTY3NDgzODkzNSwiaWF0IjoxNjc0ODM4OTM1fQ.bbYP4yevUwQ2S5e6XTaWkSq327rcKMNy0QH5w9NHm1c
Secret: QQa7LqSpu06Vmdh281JV3A
Content-Type: application/x-www-form-urlencoded
Connection: Keep-Alive
Keep-Alive: timeout=300, max=1000
content-length: 0
```
Response:
```
HTTP/1.1 200 OK
Server: nginx
Date: Fri, 27 Jan 2023 17:02:22 GMT
Content-Type: application/json; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
Strict-Transport-Security: max-age=31536000

2b
{"weather":{"main":"cloudy","temp_c":6.85}}
0
```
Request:
```
GET /emo/server/alt HTTP/1.1
Host: api.living.ai
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2NzQ4NDI1MzUsInN1YiI6ImI4ZDYxYWFhZjE2MiIsIm5iZiI6MTY3NDgzODkzNSwiaWF0IjoxNjc0ODM4OTM1fQ.bbYP4yevUwQ2S5e6XTaWkSq327rcKMNy0QH5w9NHm1c
Secret: 6hmnJ9BgO1uGgWJN4Xj3pQ
Content-Type: application/x-www-form-urlencoded
Connection: Keep-Alive
Keep-Alive: timeout=300, max=1000
content-length: 0
```
Response:
```
HTTP/1.1 200 OK
Server: nginx
Date: Fri, 27 Jan 2023 17:02:24 GMT
Content-Type: text/html; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
Vary: Accept-Encoding
Strict-Transport-Security: max-age=31536000

52
{"servers":["us-api.living.ai","eu-api.living.ai","as-api.living.ai"],"switch":""}
0
```
