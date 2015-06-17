---
layout: post
title: Ubuntu 14.04 - Python3 - matplotlib 
date: 2015-02-27
category: python
---

Ubuntu 14.04 の Python3 に matplotlib を入れた時のメモ。

Ubuntu 14.04 には最初から Python3 が入っているので、まず pip3 を入れる。

```
$ sudo apt-get install python3-pip
```

次にこれで matplotlib を入れる。

```
$ sudo pip3 install matplotlib
```

インストール自体は上手くいくのだが、実行時にエラー。

```
$ python3
>>> import matplotlib.pyplot as plt
/usr/local/lib/python3.4/dist-packages/matplotlib/backends/backend_gtk3agg.py:18: UserWarning: The Gtk3Agg backend is known to not work on Python 3.x with pycairo. Try installing cairocffi.
"The Gtk3Agg backend is known to not work on Python 3.x with pycairo. "
```

Cairocffi を入れろとあるので入れるが、コンパイル中に `ffi.h` が見つからないと言われるのでまずそっちを入れる。

```
$ sudo apt-get install libffi-dev
$ sudo apt-get install cairocffi
```

これで上手く行った。
しかしこれでも matplotlib 実行時に warning が出る。

```
/usr/local/lib/python3.4/dist-packages/matplotlib/backends/backend_gtk3.py:215: Warning: Source ID 7 was not found when attempting to remove it
GLib.source_remove(self._idle_event_id)
```

動作には問題ないが気になるので後でなんとかしたい。
