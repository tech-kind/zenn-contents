---
title: "水耕栽培キットをRaspberry PiでIoT化してみる"
emoji: "💧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["RaspberryPi", "Python", "水耕栽培"]
published: true
---

# はじめに

家庭菜園をやってみたいと思い、水耕栽培を始めて見ました。

水耕栽培は、

* 土がいらない
* 家の中で出来るので天候に左右されない
* 虫もつきにくい

といったメリットがあります。

水耕栽培を始めるにあたり、ただ始めるだけでもつまらないなと思い、せっかくなのでRasberry Piとセンサを使って、植物にとって育ちやすい環境になっているかどうかをデータとして取得してみようと考えてみました。

## 用意したもの

データ取得にあたり、用意したものは以下になります。

1. Rasberry Pi Zero WH
2. Grove Base HAT for Raspberry Pi
    - https://jp.seeedstudio.com/Grove-Base-Hat-for-Raspberry-Pi.html
3. Grove 温湿度センサ
    - https://jp.seeedstudio.com/Grove-Temperature-Humidity-Sensor-DHT11.html
4. Grove TDS水質センサ
    - https://jp.seeedstudio.com/Grove-TDS-Sensor-p-4400.html

基板はRaspberry Pi Zero WHを、センサにはGroveを使用することにしました。

Groveは専用のピンケーブルを挿すだけで使用できるツールセットで、半田付けの必要もないので誰でも簡単に扱うことのできるセンサになっています。

センサからデータを取得するためのサンプルプログラムがオープンソースでGitHubに公開されているのも特徴です。

https://github.com/Seeed-Studio/grove.py

Groveが扱うセンサ類の中から、今回は温湿度センサとTDS水質センサを使うことにします。

また、GroveセンサとRaspberry Piとの接続には専用のHAT（Grove Base HAT for Raspberry Pi）が必要で、それも合わせて用意します。

## 組み立て

組み立ててみると、こんな感じです。

![](https://storage.googleapis.com/zenn-user-upload/4a001d55402af4f87f974631.jpeg)

センサやHATを固定したかったので100均で買った画用紙と、ワイヤを使って簡単な固定をしています。

真ん中にHATがあり、温湿度センサはHATのポート12に、TDS水質センサはA0ポートに接続しています。

HATの詳細については、以下を参考にしました。

https://wiki.seeedstudio.com/jp/Grove_Base_Hat_for_Raspberry_Pi/#_3

HATの裏にRaspberry Piがあり、HATとピンで繋がっています。

最終的に、水耕栽培キットと組み合わせるとこんな感じになりました（画像が暗くて申し訳ないです）。

![](https://storage.googleapis.com/zenn-user-upload/bd1bb26986b0482f312d8b1c.jpeg)

温湿度センサは水耕栽培キットの横に、TDS水質センサの先のプローブはキットの中の水に浸かるように入れておきます。

## Raspbrry Piのセットアップ

OSのインストールやWiFiの設定等については、省略します。

Groveを使うにあたり、grove.pyをインストールする必要がありますが、最新のバージョンをインストールする場合は以下のコマンドを叩くだけでインストールを行ってくれます。便利。

``` bash
$ curl -sL https://github.com/Seeed-Studio/grove.py/raw/master/install.sh | sudo bash -s -
```

:::message
最後、インストールが完了すると以下のメッセージが表示されるので、それまでしばらく待ちましょう。

```
Successfully installed grove.py-0.6
#######################################################
Lastest Grove.py from github install complete   !!!!!
#######################################################
```

grove.pyのバージョンはインストールするタイミングによって変わります。
この記事を書いている現在は、最新のバージョンは0.6です。
:::

また、センサから読み取ったデータは「Ambient」という可視化サービスを使ってグラフ化することにしました。

https://ambidata.io/

Ambientはチャンネルを作成し、そこにデータを送るだけで勝手にグラフ化してくれるので便利です。

家で遊ぶくらいなら、自分でサーバ構築とかやるのも面倒なので、こういったサービスはとても助かります。

AmbientをPythonで利用する場合、ambient-python-libという、AmbientのPython用のライブラリをインストールしておく必要があるので、それもインストールしておきます。

``` bash
$ sudo pip install git+https://github.com/AmbientDataInc/ambient-python-lib.git
```

## ソースコード

ハードウェアの接続と、Raspberry Piのセットアップが完了したので、後はプログラムを書くだけです。

実際に書いたプログラムは以下で公開しているので、合わせてご覧ください。

https://github.com/tech-kind/smart_gardening_system

まずは温湿度センサとTDS水質センサから値を取得するクラスを用意します。

クラスの中身はそれぞれ以下のようになっています。

:::details 温湿度センサ用クラス

``` python:grove_temperature_humidity_sensor.py
import RPi.GPIO as GPIO
# from grove.helper import *
def set_max_priority(): pass
def set_default_priority(): pass
from time import sleep

GPIO.setmode(GPIO.BCM)
GPIO.setwarnings(False)

PULSES_CNT = 41


class DHT(object):
    DHT_TYPE = {
        'DHT11': '11',
        'DHT22': '22'
    }

    MAX_CNT = 320

    def __init__(self, dht_type, pin):        
        self.pin = pin
        if dht_type != self.DHT_TYPE['DHT11'] and dht_type != self.DHT_TYPE['DHT22']:
            print('ERROR: Please use 11|22 as dht type.')
            exit(1)
        self._dht_type = '11'
        self.dht_type = dht_type
        GPIO.setup(self.pin, GPIO.OUT)

    @property
    def dht_type(self):
        return self._dht_type

    @dht_type.setter
    def dht_type(self, type):
        self._dht_type = type
        self._last_temp = 0.0
        self._last_humi = 0.0

    def _read(self):
        # Send Falling signal to trigger sensor output data
        # Wait for 20ms to collect 42 bytes data
        GPIO.setup(self.pin, GPIO.OUT)
        set_max_priority()

        GPIO.output(self.pin, 1)
        sleep(.2)

        GPIO.output(self.pin, 0)
        sleep(.018)

        GPIO.setup(self.pin, GPIO.IN)
        # a short delay needed
        for i in range(10):
            pass

        # pullup by host 20-40 us
        count = 0
        while GPIO.input(self.pin):
            count += 1
            if count > self.MAX_CNT:
                # print("pullup by host 20-40us failed")
                set_default_priority()
                return None, "pullup by host 20-40us failed"

        pulse_cnt = [0] * (2 * PULSES_CNT)
        fix_crc = False
        for i in range(0, PULSES_CNT * 2, 2):
            while not GPIO.input(self.pin):
                pulse_cnt[i] += 1
                if pulse_cnt[i] > self.MAX_CNT:
                    # print("pulldown by DHT timeout %d" % i)
                    set_default_priority()
                    return None, "pulldown by DHT timeout %d" % i

            while GPIO.input(self.pin):
                pulse_cnt[i + 1] += 1
                if pulse_cnt[i + 1] > self.MAX_CNT:
                    # print("pullup by DHT timeout %d" % (i + 1))
                    if i == (PULSES_CNT - 1) * 2:
                        # fix_crc = True
                        # break
                        pass
                    set_default_priority()
                    return None, "pullup by DHT timeout %d" % i

        # back to normal priority
        set_default_priority()

        total_cnt = 0
        for i in range(2, 2 * PULSES_CNT, 2):
            total_cnt += pulse_cnt[i]

        # Low level ( 50 us) average counter
        average_cnt = total_cnt / (PULSES_CNT - 1)
        # print("low level average loop = %d" % average_cnt)

        data = ''
        for i in range(3, 2 * PULSES_CNT, 2):
            if pulse_cnt[i] > average_cnt:
                data += '1'
            else:
                data += '0'

        data0 = int(data[ 0: 8], 2)
        data1 = int(data[ 8:16], 2)
        data2 = int(data[16:24], 2)
        data3 = int(data[24:32], 2)
        data4 = int(data[32:40], 2)

        if fix_crc and data4 != ((data0 + data1 + data2 + data3) & 0xFF):
            data4 = data4 ^ 0x01
            data = data[0: PULSES_CNT - 2] + ('1' if data4 & 0x01 else '0')

        if data4 == ((data0 + data1 + data2 + data3) & 0xFF):
            if self._dht_type == self.DHT_TYPE['DHT11']:
                humi = int(data0)
                temp = int(data2)
            elif self._dht_type == self.DHT_TYPE['DHT22']:
                humi = float(int(data[ 0:16], 2)*0.1)
                temp = float(int(data[17:32], 2)*0.2*(0.5-int(data[16], 2)))
        else:
            # print("checksum error!")
            return None, "checksum error!"

        return humi, temp

    def read(self, retries = 15):
        for i in range(retries):
            humi, temp = self._read()
            if not humi is None:
                break
        if humi is None:
            return self._last_humi, self._last_temp
        self._last_humi,self._last_temp = humi, temp
        return humi, temp
```

:::

:::details TDS水質センサ用クラス

``` python:grove_tds.py
import math
import sys
import time
from grove.adc import ADC


class GroveTDS:

    def __init__(self, channel):
        self.channel = channel
        self.adc = ADC()

    @property
    def TDS(self):
        value = self.adc.read(self.channel)
        if value != 0:
            voltage = value*5/1024.0
            tdsValue = (133.42/voltage*voltage*voltage-255.86*voltage*voltage+857.39*voltage)*0.5
            return tdsValue
        else:
            return 0
```

:::

これらのクラスを使って、1分置きにセンサからデータを取得します。

作成したプログラムが以下になります。

``` python:gardening_system.py
#!/usr/bin/env python3
import ambient
import sys
import time
import datetime
import schedule

from sensor.grove_tds import GroveTDS
from sensor.grove_temperature_humidity_sensor import DHT

DHT_TYPE = "11"
DHT_PIN = 12
TDS_CHANNEL = 0
AMBIENT_CHANNEL_ID = "-----"
AMBIENT_WRITE_KEY = "----------------"


def get_sensor_info():
    humi, temp = dht_sensor.read()

    data = {
        "created": datetime.datetime.now().strftime("%Y-%m-%d%H:%M:%S"),
        "d1": temp,
        "d2": humi,
        "d3": round(tds_sensor.TDS, 2)
    }
    try:
        am.send(data, timeout=10.0)
    except requests.exceptions.RequestException:
        pass


def main():
    schedule.every(1).minutes.do(get_sensor_info)

    while True:
        schedule.run_pending()
        time.sleep(1)


if __name__ == '__main__':
    dht_sensor = DHT(DHT_TYPE, DHT_PIN)
    tds_sensor = GroveTDS(TDS_CHANNEL)
    am = ambient.Ambient(AMBIENT_CHANNEL_ID, AMBIENT_WRITE_KEY)
    main()
```

「DHT_TYPE」はセンサの種類を指定しています。

温湿度センサにはDHT11とDHT22があるのですが、今回はDHT11を使用しているので`"11"`を指定しています。

「DHT_PIN」はセンサがHATとどのポートに繋がっているかによって変わります。

利用する際は繋がっているポートに適宜読み替えてください。

また、「AMBIENT_CHANNEL_ID」と「AMBIENT_WRITE_KEY」はAmbientでチャンネルを作成したときに、発行されるIDとキーが実際には入ります。

main関数の中で、1分置きにget_sensor_info関数が実行されるようにスケジュール登録を行っています。

get_sensor_info関数はセンサからデータを読み取り、Ambientにデータを送信しています。

実際に、プログラムを動かしてみましょう。

上手くいけば、Ambientの自分が作成したチャンネルに自動的にデータが送られグラフが更新されるはずです。

僕の場合は、こんな感じです。いいですね。

![](https://storage.googleapis.com/zenn-user-upload/a6365b890c409884608bd285.jpg)

## 常駐化

最後に、作成したプログラムを自動的に起動させるためにsystemdを使って常駐化させます。

systemdにサービスを登録するには、serviceファイルの作成が必要になるので、vim等のエディタを使って以下のフォルダにserviceファイルを作成します。

``` bash
sudo vim /etc/systemd/system/gardening-system.service
```

今回は、「gardening-system」というserviceファイルにしていますが、自分でわかりやすい名前をつけてください。

ファイルの中には以下を記述します。

``` ini
[Unit]
Description=systemd-test daemon
[Service]
ExecStart=/usr/bin/python3 /home/pi/gardening_system.py
Restart=no
Type=simple
[Install]
WantedBy=multi-user.target
```

`ExecStart`で実際に動かしたいプログラムを指定します。

僕の場合は、「gardening_system.py」というファイルを、「/home/pi」ディレクトリに入れているので上のようになっていますが、

実際には自身が動かしたいプログラムのファイルパスを絶対パスで指定します。

後は、`systemctl`コマンドを使うことで、プログラムがサービスとして起動してくれます。

``` bash
$ sudo systemctl enable sysd-check.service
$ sudo systemctl start sysd-check.service
```

これで、Raspberry Piが再起動しても自動的にサービスが起動してくれます。便利。

## 最後に

今回は、Raspberry Piを使って簡単なIoTを行ってみました。

水質がわかるとどのくらい肥料を入れたらよいのかがわかりやすくなりました。

ゆくゆくは、自動水やりとかも出来るようになったらいいですね。