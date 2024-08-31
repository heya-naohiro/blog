+++
title = "paho mqtt over websocketにおけるkeepalive"
date = 2024-08-31
draft = false
+++

## やったこと
MQTT over websocketの通信のkeepaliveを見る

## 構成
- サーバ mochi/serverのwebsocket exampleを使う <https://github.com/mochi-mqtt/server/tree/main/examples/websocket>
- クライアントはpahoを使う <https://github.com/eclipse/paho.mqtt.golang>

- ソースコードは下記の改変
<https://qiita.com/emqx_japan/items/31472d22e9a2cad36dd8>

```go:client.go
package main

import (
	"fmt"
	"log"
	"os"
	"time"

	mqtt "github.com/eclipse/paho.mqtt.golang"
)

var messagePubHandler mqtt.MessageHandler = func(client mqtt.Client, msg mqtt.Message) {
	fmt.Printf("Received message: %s from topic: %s\n", msg.Payload(), msg.Topic())
}

var connectHandler mqtt.OnConnectHandler = func(client mqtt.Client) {
	fmt.Println("Connected")
}

var connectLostHandler mqtt.ConnectionLostHandler = func(client mqtt.Client, err error) {
	fmt.Printf("Connect lost: %v", err)
}

func main() {
	mqtt.ERROR = log.New(os.Stdout, "[ERROR] ", 0)
	mqtt.CRITICAL = log.New(os.Stdout, "[CRIT] ", 0)
	mqtt.WARN = log.New(os.Stdout, "[WARN]  ", 0)
	mqtt.DEBUG = log.New(os.Stdout, "[DEBUG] ", 0)
	var broker = "192.168.0.100"
	var port = 1882
	opts := mqtt.NewClientOptions()
	opts.AddBroker(fmt.Sprintf("ws://%s:%d", broker, port))
	opts.SetDefaultPublishHandler(messagePubHandler)
	opts.OnConnect = connectHandler
	opts.OnConnectionLost = connectLostHandler
	opts.KeepAlive = 5

	client := mqtt.NewClient(opts)
	if token := client.Connect(); token.Wait() && token.Error() != nil {
		panic(token.Error())
	}
	time.Sleep(time.Second * 120)
}

```

### Wireshark result
#### Connect packet
```
WebSocket
    1... .... = Fin: True
    .000 .... = Reserved: 0x0
    .... 0010 = Opcode: Binary (2)
    1... .... = Mask: True
    .000 1110 = Payload length: 14
    Masking-Key: 2299147f
    Masked payload
    Payload
Data (14 bytes)
    Data: 100c00044d515454040200050000
    [Length: 14]

```

Websocketの形式でMaskされており、中身は10から始まる。1はMQTT fixed HeaderのConnectを示している。

### Pingreq / Pingres

Client -> Host
```
WebSocket
    1... .... = Fin: True
    .000 .... = Reserved: 0x0
    .... 0010 = Opcode: Binary (2)
    1... .... = Mask: True
    .000 0010 = Payload length: 2
    Masking-Key: 75e2bbfd
    Masked payload
    Payload
Data (2 bytes)
    Data: c000
    [Length: 2]
```
cはPINGREQ

Host -> Client
```
WebSocket
    1... .... = Fin: True
    .000 .... = Reserved: 0x0
    .... 0010 = Opcode: Binary (2)
    0... .... = Mask: False
    .000 0010 = Payload length: 2
    Payload
Data (2 bytes)
    Data: d000
    [Length: 2]

```
dはPINGRESP

MQTT Headerについては下記のリンクを参照
<https://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.html#_Toc398718020>


## わかったこと

- MQTT over WebsocketはWebsocketのバイナリに乗せてMQTTのプロトコルを流す。
- WebsocketはMaskedという概念があり、not tlsであってもパケットがそのまま流れるわけではない。（WiresharkはUnmaskして表示してくれる）
- paho mqtt go clientのkeepaliveはWebsocketのKeepalive()ではなく,MQTTのPingreq/Pingresを利用する。
- WebsocketにもPing/Pongの仕組みがあるが、今回確認できなかった。規格：<https://datatracker.ietf.org/doc/html/rfc6455#section-5.5.2>
