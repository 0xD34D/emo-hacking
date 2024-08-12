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

**`Request:`**
```
GET /time?tz=America/Los_Angeles HTTP/1.1
Host: api.living.ai
Connection: close
Secret: 
content-length: 0
```
**`Response:`**
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

**`Request:`**
```
GET /token/b8d61aaaf162 HTTP/1.1
Host: api.living.ai
Connection: close
Secret: igZ_haD7wuHeMszUE8RBNQ
content-length: 0
```
**`Response:`**
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

**`Request:`**
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
**`Response:`**
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
**`Request:`**
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
**`Response:`**
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
**`Request:`**
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
**`Response:`**
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
**`Request:`**
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
**`Response:`**
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
**`Request:`**
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
**`Response:`**
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
#### inside an OTA packet
**`Request:`**
```
GET https://api.living.ai/emo/ota/res/28 HTTP/1.1
Host: api.living.ai
Connection: close
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE3MjMyMzM5MjcsInN1YiI6ImI4ZDYxYWFhZjE2MiIsIm5iZiI6MTcyMzIyNjMyMSwiaWF0IjoxNzIzMjI2MzIxLCJ2ZXJzaW9uIjoiMjgiLCJuYW1lIjoiIn0.qkzNnBACFAMLicCyb-RP1PgDDjsMdaJnrNPULubM0CI
Secret: p1HaacpV_p5yzs5C0svmXg
```
**`Response:`**
```
HTTP/1.1 200 OK
Server: nginx
Date: Fri, 09 Aug 2024 17:58:46 GMT
Content-Type: application/json; charset=utf-8
Transfer-Encoding: chunked
Connection: close
Strict-Transport-Security: max-age=31536000
```
**`JSON payload:`**
```json
{
    "updates": {
        "esp32": "sdcard/esp32/alexa.bin",
        "esp_sha256": "",
        "esp_sum": 0,
        "host": "res-us-east-1.living.ai",
        "hosts": {
            "0": "res.living.ai",
            "1": "eu-res.living.ai",
            "2": "res.livingai.cn"
        },
        "k210": "sdcard/k210/2021_01_30",
        "k210_sha256": "",
        "k210_sum": 411,
        "model": {
            "0": "/model/"
        },
        "sd": {
            "avi": {
                "0": "/avi_y4564/",
                "1": "ttt_please_say_a_number.avi",
                "10": "react_to_obs_13.avi",
                "11": "loud_sound2.avi",
                "12": "loud_sound1.avi",
                "13": "gohome_connect.avi",
                "14": "emo_play_a_game_start.avi",
                "15": "emo_play_a_game_loop1.avi",
                "16": "emo_play_a_game_end.avi",
                "17": "Daily_Read_a_book_start.avi",
                "18": "Daily_Read_a_book_loop1.avi",
                "19": "Daily_Read_a_book_end.avi",
                "2": "touch_xing_start.avi",
                "20": "not_execute.avi",
                "21": "chatgpt_end.avi",
                "22": "chatgpt_hear.avi",
                "23": "chatgpt_speak.avi",
                "24": "chatgpt_start.avi",
                "25": "chatgpt_stay.avi",
                "26": "show_painting2.avi",
                "27": "Partygoer_eixt.avi",
                "28": "emo_maraca_ready1.avi",
                "29": "emo_piano_1.avi",
                "3": "touch_xing_loop.avi",
                "30": "emo_piano_2.avi",
                "31": "emo_piano_eixt1.avi",
                "32": "emo_piano_ready1.avi",
                "33": "emo_tambourine_1.avi",
                "34": "emo_tambourine_eixt1.avi",
                "35": "emo_tambourine_ready1.avi",
                "36": "emo_trumpet_1.avi",
                "37": "emo_trumpet_eixt1.avi",
                "38": "emo_trumpet_reday1.avi",
                "39": "emo_maraca_eixt1.avi",
                "4": "touch_xing_end.avi",
                "40": "Partygoer_idle1.avi",
                "41": "Partygoer_loop1.avi",
                "42": "Partygoer_loop2.avi",
                "43": "Partygoer_loop3.avi",
                "44": "Partygoer_loop4.avi",
                "45": "Partygoer_ready.avi",
                "46": "Partygoer_voice_wait.avi",
                "47": "Partygoer_voice_wake_up.avi",
                "48": "speaker_con_fail.avi",
                "49": "speaker_con_succ.avi",
                "5": "speak_4s_1.avi",
                "50": "speaker_connecting.avi",
                "51": "DJ1_Vdown.avi",
                "52": "DJ_voice_wake_up.avi",
                "53": "DJ_voice_wake_up2.avi",
                "54": "DJ1_exit.avi",
                "55": "DJ1_idle1.avi",
                "56": "DJ1_idle2.avi",
                "57": "DJ1_loop1.avi",
                "58": "DJ1_loop2.avi",
                "59": "DJ1_next.avi",
                "6": "show_painting_png.avi",
                "60": "DJ1_play.avi",
                "61": "DJ1_ready.avi",
                "62": "DJ1_stop.avi",
                "63": "DJ_voice_wait.avi",
                "64": "DJ1_Vup.avi",
                "65": "DJ2_in.avi",
                "66": "DJ2_loop1.avi",
                "67": "DJ2_loop2.avi",
                "68": "DJ2_loop3.avi",
                "69": "DJ2_out.avi",
                "7": "show_painting.avi",
                "70": "emo_drum_eixt1.avi",
                "71": "emo_drum_ready1.avi",
                "72": "emo_ensemble_drum_1.avi",
                "73": "emo_ensemble_maraca_1.avi",
                "74": "emo_ensemble_piano_1.avi",
                "75": "santasong2023_1.avi",
                "76": "santasong2023_2.avi",
                "77": "sings1_1.avi",
                "78": "sings2_1.avi",
                "8": "shake_loop_5.avi",
                "9": "shake_end_8.avi"
            },
            "json": {
                "0": "/json/",
                "1": "profile.json"
            },
            "mot": {
                "0": "/mot/",
                "1": "ttt_please_say_a_number.mot",
                "10": "react_to_obs_13.mot",
                "11": "shake_end_8.mot",
                "12": "show_painting.mot",
                "13": "speak_4s_1.mot",
                "14": "touch_xing_end.mot",
                "15": "touch_xing_loop.mot",
                "16": "touch_xing_start.mot",
                "17": "new_years_day.mot",
                "18": "d4_Click1.mot",
                "19": "DJ1_exit.mot",
                "2": "Group_photo_gesture_3.mot",
                "20": "DJ1_ready.mot",
                "21": "emo_ensemble_drum_1.mot",
                "22": "santasong2023_1.mot",
                "23": "santasong2023_2.mot",
                "24": "sings1_1.mot",
                "25": "sings1_2.mot",
                "26": "sings1_3.mot",
                "27": "sings1_4.mot",
                "28": "sings1_5.mot",
                "29": "sings2_1.mot",
                "3": "d7_FlashBack.mot",
                "30": "sings2_2.mot",
                "31": "sings2_3.mot",
                "32": "sings2_4.mot",
                "4": "emo_play_a_game_end.mot",
                "5": "emo_play_a_game_loop1.mot",
                "6": "emo_play_a_game_start.mot",
                "7": "gohome_connect.mot",
                "8": "loud_sound1.mot",
                "9": "loud_sound2.mot"
            },
            "mp3": {
                "0": "/mp3_m3456/",
                "1": "ttt_please_say_a_number_000.mp3",
                "10": "speak_tohome_goodbye_000.mp3",
                "11": "speak_notconnect_homestation_000.mp3",
                "12": "show_painting_000.mp3",
                "13": "shake_end_8_000.mp3",
                "14": "react_to_obs_13_000.mp3",
                "15": "not_execute4_000.mp3",
                "16": "not_execute3_000.mp3",
                "17": "not_execute2_000.mp3",
                "18": "loud_sound2_000.mp3",
                "19": "loud_sound1_000.mp3",
                "2": "touch_xing_start_000.mp3",
                "20": "gohome_connect_000.mp3",
                "21": "emo_play_a_game_start_000.mp3",
                "22": "emo_play_a_game_loop1_000.mp3",
                "23": "emo_play_a_game_end_000.mp3",
                "24": "not_execute_000.mp3",
                "25": "chatgpt_end_000.mp3",
                "26": "chatgpt_hear_000.mp3",
                "27": "chatgpt_start_000.mp3",
                "28": "emo_maraca_ready1_000.mp3",
                "29": "speaker_connecting_000.mp3",
                "3": "touch_xing_loop_000.mp3",
                "30": "speaker_con_succ_000.mp3",
                "31": "speaker_con_fail_000.mp3",
                "32": "Partygoer_ready_000.mp3",
                "33": "Partygoer_eixt_000.mp3",
                "34": "emo_trumpet_1_000.mp3",
                "35": "emo_tambourine_ready1_000.mp3",
                "36": "emo_tambourine_1_000.mp3",
                "37": "emo_piano_ready1_000.mp3",
                "38": "emo_piano_eixt1_000.mp3",
                "39": "emo_piano_2_000.mp3",
                "4": "touch_xing_end_000.mp3",
                "40": "emo_piano_1_000.mp3",
                "41": "emo_ensemble_piano_1_000.mp3",
                "42": "emo_ensemble_maraca_1_000.mp3",
                "43": "DJ_voice_wake_up_000.mp3",
                "44": "emo_ensemble_drum_1_000.mp3",
                "45": "emo_drum_ready1_000.mp3",
                "46": "DJ1_Vup_000.mp3",
                "47": "DJ1_Vdown_000.mp3",
                "48": "DJ1_stop_000.mp3",
                "49": "DJ1_ready_000.mp3",
                "5": "speak_tohome_willmissyou_000.mp3",
                "50": "DJ1_play_000.mp3",
                "51": "DJ1_next_000.mp3",
                "52": "DJ1_exit_000.mp3",
                "53": "sings1_10_sad.mp3",
                "54": "sings2_11_see_you_again.mp3",
                "55": "sings2_10_shi_jie_bei_1998.mp3",
                "56": "sings2_9_duo_la_A_meng2.mp3",
                "57": "sings2_8_duo_la_A_meng1.mp3",
                "58": "sings2_7_tuziwu.mp3",
                "59": "sings2_6_tom.mp3",
                "6": "speak_tohome_willbeback_000.mp3",
                "60": "sings2_5_Orpheus_in_the_Underworld.mp3",
                "61": "sings2_4_lian_ai_xun_huan.mp3",
                "62": "sings2_3_jingle_bell.mp3",
                "63": "sings2_2_huan_le_song.mp3",
                "64": "sings2_1_baby.mp3",
                "65": "santasong2023_1.mp3",
                "66": "sings1_9_fall_in_love_for_the_night.mp3",
                "67": "sings1_8_bao_ke_meng.mp3",
                "68": "sings1_7_foxsay.mp3",
                "69": "sings1_6_wallerman.mp3",
                "7": "speak_tohome_waitforme_000.mp3",
                "70": "sings1_5_shape_of_you.mp3",
                "71": "sings1_4_love_the_way_you_lie.mp3",
                "72": "sings1_3_guan_lan_gao_shou.mp3",
                "73": "sings1_2_dance_monkey.mp3",
                "74": "sings1_1_ballroom.mp3",
                "75": "santasong2023_2.mp3",
                "8": "speak_tohome_takecare_000.mp3",
                "9": "speak_tohome_seeyoulater_000.mp3"
            }
        }
    },
    "version-name": "2.5.0",
    "version-num": 33
}
```