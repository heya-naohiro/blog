+++
title = "RustでMQTTブローカー作ってみたら世界が広がった"
date = 2024-04-07
+++

- [講演動画：大きな問題のほうが小さな問題より解くのは簡単だ / Blue Whale Systems株式会社 植山 類氏](https://youtu.be/C4uKuNTMmcU?si=9muNve_fyiP5Y1IZ)
- [書籍：世界一流エンジニアの思考法 | 牛尾 剛](https://amzn.to/3VJP5Ow)
の２つから物事を正面から取り組む勇気を手に入れた。そして、業務でよく使っているプロトコル,MQTTを扱うアプリケーション、MQTTブローカーを実装してみようと思った。それが出来るのでは？と感じさせる公式チュートリアルがとても良かった。
- [Rust the book Chapter 20: Final Project: Building a Multithreaded Web Server](https://doc.rust-lang.org/book/ch20-00-final-project-a-web-server.html)
→ HTTPって実装できるんだ...
- [Tokio tutorial](https://tokio.rs/tokio/tutorial)
→ Redisのプロトコルって実装できるんだ...
→MQTTを実装してみるか...
- [mqtt broker](https://github.com/heya-naohiro/mqtt-server) 

実際に仕様とにらめっこしながら実装を進めると世界が広がってすこしモヤモヤが晴れた。自己効力感が高まった。自分が普段使っている何か、と同じものを１から自分で作ってみるという体験からしか得られない栄養素があった。
　MQTTは簡単なプロトコルだとも言われていますが、フルフルで実装しようとすると面倒な感じはします。本質的には非同期が難しいといった感じがします。でも、ある程度規格に準拠するものを作ると世界と繋がれる感覚がして"インターネット"良い