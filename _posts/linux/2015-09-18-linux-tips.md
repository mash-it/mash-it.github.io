---
layout: post
title: Linux Tips
date: 2015-09-15
category: linux
---

単独ページを作るのも面倒な雑多な話題。

# ファイル転送

サーバーとのファイル転送は scp が一般的だが、暗号化のせいで転送速度が落ちる場合がある。ローカルネットワークなど暗号化強度がそれほど求められないシーンでは、高速な暗号化方式を使ったり、そもそも暗号化しないという方法がある。

暗号化方式は `scp -c` オプションで設定できる。arcfour とかが結構速い。

```
$ scp -c arcfour myserver:xxxx/xxxx.dat .
```

暗号化しない場合は rcp がある.

```
$ rcp cyrus:xxxx/xxxx.dcd .
```

Gigabit Ethernet を使っている場合、規格上の最高速度は 1Gbps = 128MB/s となる。半分くらい出ていれば問題ないと思うが、10Mb/s とかだと改善の努力が欲しい。

# PDF編集

2個のPDFファイルを結合するツールとか知ってると地味に便利かもしれない.

```
$ sudo apt-get install pdftk
$ pdftk a.pdf b.pdf cat output ab.pdf
```

# 日本語入力

Linux の日本語入力システムは何種類かあるが fcitx-mozc が多分一番良い。fcitx とは中国で開発されているインプットメソッドフレームワークで、mozc は Google 日本語入力から派生した IME である。Ubuntu に標準で入ってる iBus とかいうインプットメソッドフレームワークは漢字変換という思想を解さない欧米人が開発した Dammit なシステムであるため、ここはアジア人同士わかりあえる fcitx を使うとよい。

インストールし設定画面を開く

```
$ sudo apt-get install fcitx-mozc
$ fcitx-configtools
```

設定画面で Input Method に Mozc を追加。

あとは System Settings -> Language Support -> Keyboard Input Method System を iBus から fcitx に変更して再起動。






