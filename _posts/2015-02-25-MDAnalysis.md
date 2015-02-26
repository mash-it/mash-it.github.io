---
layout: post
title: MDAnalysis
date: 2015-02-15
---

MDAnalysis は Python で各種 MD Trajectory の結果を解析するツール。

CHARMM, GROMACS, NAMD, Amber と幅広く対応している。

# Install

pip で入手可能。

```
$ sudo pip install MDAnalsis
```

# Trajectory から座標を取得

まず Trajectory 全体を Universe というクラスのオブジェクトに格納する。
Trajectory は2個のファイル (psf/dcd, gro/trr など) からなる。

{% highlight python %}
from MDAnalysis import Universe
univ = Universe("traj.psf", "traj.dcd")
{% endhighlight %}

Universe クラスには atoms というプロパティがある。atoms は AtomGroups と呼ばれるクラスで coordinates() というメソッドを実行するとN個原子の座標をN行3列の numpy.array 形式で返す。

{% highlight python %}
print univ.atoms.coordinates()
{% endhighlight %}

AtomGroups クラスには他にも totalMass(), centerOfMass(), principleAxes(), radiusOfGyration() など座標に関するメソッドが提供されている。

# フレームを変更

MD Trajectory の座標データは、3次元 x 原子数 x 時間(フレーム) という3次元配列からなる。座標を取得するフレームは以下のようにかなり珍妙な（Pythonらしからぬ）方法で指定する。

{% highlight python %}
# univ の20フレーム目に移動
univ.trajectory[20]
# 20フレーム目の座標を取得
print univ.atoms.coordinates()
{% endhighlight %}

このように、配列の要素を呼び出す要領でフレームの位置を切り替える。 おそらく内部でイテレータとかジェネレータとかそういうものを使っているため、こういう記法になるのだと思われる。

同様に for 構文を使って各フレームのデータを取り出すことが出来る。

{% highlight python %}
# 各フレームの重心を表示
for ts in univ.trajectory:
	print univ.atoms.centerOfMass()
{% endhighlight %}

# 原子団を選択

いちいち全部の原子を扱うのは面倒なので、一部の原子を選択する。これは Universe の selectAtoms() というメソッドを用いる。たとえば1番の残基のみを選択したい場合

{% highlight python %}
residue1 = univ.selectAtoms("resid 1")
{% endhighlight %}

と書く。ほかにも

{% highlight python %}
univ.selectAtoms("resid 1:10")		# 1番から10番の残基
univ.selectAtoms("resname LYS")		# Lys残基
univ.selectAtoms("name CA")		# Ca原子
univ.selectAtoms("bynum 1-6")		# 原子の番号
{% endhighlight %}

これ以外の選択方法については [こちら](http://mdanalysis.googlecode.com/svn/trunk/doc/html/documentation_pages/selections.html) 。
これらは戻り値が AtomGroups クラスであるため、先に挙げた coordinates() などのメソッドが使える。
