+++
title = "mqtt-stresserで負荷試験"
date = 2024-04-13
+++

## Go製のMQTTBroker用テスト: mqtt-stresserの紹介
[mqtt-stresser](https://github.com/inovex/mqtt-stresser)

シンプルにbuildしてrunするだけ。今回はtargetを[mochi-co](https://github.com/mochi-mqtt/server)
```bash
$ go build .
```

なにも指定しないとPayloadには"this is msg #1"のような値が使われるようです。
```go
func defaultPayloadGen() PayloadGenerator {
	return func(i int) string {
		return fmt.Sprintf("this is msg #%d!", i)
	}
}
```

```bash
[~/mqtt-stresser]$./mqtt-stresser -broker tcp://localhost:1883 -num-clients 10 -num-messages 150 -rampup-delay 1s -rampup-size 10 -global-timeout 180s -timeout 20s
10 worker started
..................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................
# Configuration
Concurrent Clients: 10
Messages / Client:  1500

# Results
Published Messages: 1500 (100%)
Received Messages:  1500 (100%)
Completed:          10 (100%)
Errors:             0 (0%)

# Publishing Throughput
Fastest: 55175 msg/sec
Slowest: 20228 msg/sec
Median: 43892 msg/sec

  < 23722 msg/sec  10%
  < 37701 msg/sec  20%
  < 41196 msg/sec  30%
  < 44691 msg/sec  60%
  < 48185 msg/sec  70%
  < 51680 msg/sec  80%
  < 55175 msg/sec  90%
  < 58669 msg/sec  100%

# Receiving Througput
Fastest: 202375 msg/sec
Slowest: 17580 msg/sec
Median: 23824 msg/sec

  < 36059 msg/sec  90%
  < 220854 msg/sec  100%
```