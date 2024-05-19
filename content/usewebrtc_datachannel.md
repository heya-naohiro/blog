+++
title = "webrtcのdatachannelを触ってみる"
date = 2024-05-18
draft = false
+++

## 目的
WebRTCでデータチャネルで双方向の通信を手を動かしてP2Pでのデータ通信を体感する

## WebRTCについて参考記事
- [WebRTC コトハジメ](https://gist.github.com/voluntas/67e5a26915751226fdcf)
- [WebRTC徹底解説](https://zenn.dev/yuki_uchida/books/c0946d19352af5)
- [ブロックチェーンのP2P通信にWebRTCを使う](https://qiita.com/erukiti/items/ec872816eb3a8415f0cd)
- [webrtc.rs](https://github.com/webrtc-rs/webrtc) (プロトコルスタックの図がわかりやすい)

## 構成
- A: [Webrtc.rs](https://github.com/webrtc-rs/webrtc) on Raspberry Pi
- B: Next.js/React on Chrome
- C: [TURN: Cloudflare (beta)](https://developers.cloudflare.com/calls/turn/)
## CloudflareでTURN Serviceを作成
Calls > Create: Tern Service Token
{{ image(path="/2024-05-19.png") }}

## A: Rustで書いたアプリケーション 
[webrtc.rsのExample](https://github.com/webrtc-rs/webrtc/blob/master/examples/examples/data-channels-create/data-channels-create.rs)
を参考に、RTCIceServerの箇所をCloudflareのものに変更する(トークンを払い出された際に表示されているもの)
```
ice_servers: vec![RTCIceServer {
            urls: vec!["stun:stun.l.google.com:19302".to_owned()],
            ..Default::default()
        }],
```

> ICE (Interactive Connectivity Establishment)とはNAT超えのためにSTUN/TURNのプロトコル選択のための仕組み

## B: Browser で書いたアプリケーション
参考：https://jsfiddle.net/swgxrp94/20/
```typescript
"use client"

import { useEffect } from "react";
import { useState } from 'react';

export default function Home() {
  const [data, setData] = useState('no data');
  const [localsdp, setLocalDescription] = useState('');
  const [dataChannel, setDataChannel] = useState<RTCDataChannel | null>(null);
  const [peer, setPeerConnection] = useState<RTCPeerConnection | null>(null);

  const REPLACE_WITH_USERNAME = "";
  const REPLACE_WITH_CREDENTIAL = "";
  const config = {
    iceServers: [
      {
        urls: "stun:stun.cloudflare.com:3478",
      },
      {
        urls: "turn:turn.cloudflare.com:3478",
        username: REPLACE_WITH_USERNAME,
        credential: REPLACE_WITH_CREDENTIAL,
      },
      {
        urls: "turns:turn.cloudflare.com:443?transport=tcp",
        username: REPLACE_WITH_USERNAME,
        credential: REPLACE_WITH_CREDENTIAL,
      },
      {
        urls: "turn:turn.cloudflare.com:3478?transport=tcp",
        username: REPLACE_WITH_USERNAME,
        credential: REPLACE_WITH_CREDENTIAL,
      },
    ],
  };
  useEffect(() => {
    let peer = new RTCPeerConnection(config);
    peer.onsignalingstatechange = (e) => {
      console.log(peer.signalingState);
      setLocalDescription(btoa(JSON.stringify(peer.localDescription)));
    };
    peer.oniceconnectionstatechange = (e) => {
      console.log(e);
      setLocalDescription(btoa(JSON.stringify(peer.localDescription)));
    };
    peer.ondatachannel = (e) => {
      let dc = e.channel
      dc.onclose = () => console.log('dc has closed');
      dc.onopen = () => console.log('dc has opened');
      dc.onmessage = e => {
        setData(e.data);
        console.log(`Message from DataChannel '${dc.label}' payload '${e.data}'`);
      }
      setDataChannel(dc);
    };

    console.log(peer);
    setPeerConnection(peer);
  }, []);

  useEffect(() => {
    console.log("peer change");
    console.log(peer?.localDescription);
    console.log(btoa(JSON.stringify(peer?.localDescription)));
    setLocalDescription(btoa(JSON.stringify(peer?.localDescription)));
  }, [peer]);

  function handleSdp(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const form = e.currentTarget;
    const formData = new FormData(form);
    const formJson = Object.fromEntries(formData.entries());
    if (formJson['sdp'] === '') {
      return
    }
    const remoteSdp = formJson['sdp'] as string;
    console.log('remotesdp: ' + remoteSdp);
    peer?.setRemoteDescription(new RTCSessionDescription(JSON.parse(atob(remoteSdp))))
    peer?.createAnswer().then(d => {
      peer.setLocalDescription(d);
      setLocalDescription(btoa(JSON.stringify(peer.localDescription)));
    });
  };

  function handleSendData(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const form = e.currentTarget;
    const formData = new FormData(form);
    const formJson = Object.fromEntries(formData.entries());
    if (formJson['send'] === '') {
      return
    }
    const senddata = formJson['send'] as string;
    dataChannel?.send(senddata);

  }

  return (
    <main className="flex min-h-screen flex-col items-center justify-between p-24">
      <p>hello world</p>
      <div>
        <form method="post" onSubmit={handleSdp}>
          SDP :<textarea name="sdp" />
          <button type="submit">Submit form</button>
        </form>
      </div>
      <div>{localsdp}</div>
      <button onClick={() => global.navigator.clipboard.writeText(localsdp)}>Copy SDP to Clipboard</button>
      <div>{data}</div>

      <form method="post" onSubmit={handleSendData}>
        SendData :<input name="send" defaultValue="" />
        <button type="submit">Send data</button>
      </form>

    </main >
  );
}


```

### やってること（順不同）

1. 初期化
```typescript
    let peer = new RTCPeerConnection(config);
```
2. Data channelの作成: データチャネルを作成し、React stateに入れておく & データを受けたときの処理を定義する
```typescript
peer.ondatachannel = (e) => {
      let dc = e.channel
      dc.onclose = () => console.log('dc has closed');
      dc.onopen = () => console.log('dc has opened');
      dc.onmessage = e => {
        setData(e.data);
        console.log(`Message from DataChannel '${dc.label}' payload '${e.data}'`);
      }
      setDataChannel(dc);
    };
```

3. データチャネルに送信する
```typescript
function handleSendData(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const form = e.currentTarget;
    const formData = new FormData(form);
    const formJson = Object.fromEntries(formData.entries());
    if (formJson['send'] === '') {
      return
    }
    const senddata = formJson['send'] as string;
    dataChannel?.send(senddata);

  }
```


4. Remote SDPを入力/local SDPを表示するインターフェイスを用意
```typescript
function handleSdp(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const form = e.currentTarget;
    const formData = new FormData(form);
    const formJson = Object.fromEntries(formData.entries());
    if (formJson['sdp'] === '') {
      return
    }
    const remoteSdp = formJson['sdp'] as string;
    console.log('remotesdp: ' + remoteSdp);
    peer?.setRemoteDescription(new RTCSessionDescription(JSON.parse(atob(remoteSdp))))
    peer?.createAnswer().then(d => {
      peer.setLocalDescription(d);
      setLocalDescription(btoa(JSON.stringify(peer.localDescription)));
    });
  };
```

## SDP(Session Description Protocol)の交換について
P2Pで通信するために互いにSDPを交換する必要がある。今回はコピペで交換する。
- A:Rustを起動して表示されるSDPをコピー
- B:Browserに入力して、表示されるSDPをコピー
- A:Rustに入力

実際の製品ではネットワーク経由でもっと凝った方法をでExchangeして経路とプロトコルを決定する。


## Result
左がブラウザ、右がRaspi

{{ image(path="/result_send.jpg") }}

