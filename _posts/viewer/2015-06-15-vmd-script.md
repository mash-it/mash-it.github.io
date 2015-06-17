---
layout: post
title: VMD script
date: 2015-06-25
categories: viewer
permalink: /:categories/:title
---

ターミナルでVMDを起動するとウィンドウが開くとともにターミナルに

```
vmd >
```

というコンソールが現れる。ここに VMD スクリプトを入力していく。

## 基本

VMD スクリプトは Tcl スクリプトの上に実装されている。
骨格は bash と大体一緒。

変数に値を代入

```
> set x 10
```

値を表示

```
> put $x
10
```

中括弧で囲むとベクトルになり、加算や [その他の演算](http://www.ks.uiuc.edu/Research/vmd/vmd-1.7.1/ug/node163.html) ができる。

```
> set x {1 2 3}
> set y {4 5 6}
> vecadd $x $y
5.0 7.0 9.0
```

関数は `proc` で定義。

```
proc myfunc {} {
    puts "OK"
}

myfunc          # OK と表示される
```

引数付き関数は

```
proc myfunc {x} {
    puts $x
}

myfunc "OK"      # OK と表示される
```

あらかじめ書いたスクリプトを VMD 起動時に読み込む

```
$ vmd -e script.tcl
```

スクリプトを起動後に読み込む

```
vmd > source script.tcl
```

## 分子構造をインポート

保存した PDB ファイルを開く

```
vmd > mol new 1ubq.pdb
```

PDB ID を指定してサーバーから PDB を拾ってくる

```
vmd > mol pdbload 1UBQ
```

開いた分子にはそれぞれ `molID` が付与される。VMD Main のウィンドウで確認可能。

Trajectory では座標ファイルとトポロジーファイルの2個を要求される場合が多い。たとえば DCD + PSF の場合は以下のように開く。

```
vmd > mol new traj.dcd
vmd > mol addfile traj.psf
```

## 画面操作

ウィンドウのサイズを設定。
動画にレンダリングするときはこのサイズになるので、使い方を想定したウィンドウサイズを予め指定しておこう。

```
display resize 640 480
```

背景色を指定

```
color Display Background white
```

画面全体を移動

```
translate to 0 0 0		# 指定した位置に並進移動
translate by 1 0 0 		# x軸方向に1だけ並進移動
rotate x by 45			# x軸方向に45度だけ回転
scale by 2				# 画面全体を2倍に拡大
rock x by 5 9			# 揺らす
rock off				# 揺れを止める
```

## 分子の Representation 操作

VMD では分子内にさらに Rep が存在し、個別に色などを指定できる。
分子内の複数ドメインを色分けしたい場合などは、それぞれに Rep を指定する必要がある。

mol 0 の Rep の一覧を表示 (この時点では selection:all の Rep 0 しか存在しない)

```
vmd > mol list 0
Status of molecule 1ubq:
1ubq  Atoms:660  Frames (C): 1(0)  Status:ADfT
Atom representations: 1
0:  on, 660 atoms selected.
  Coloring method: Name
   Representation: Lines
        Selection: all
```

mol 0 の 1-20 番残基を新たな Rep にする (すでに Rep 0があるので、新たに Rep 1 ができる)

```
vmd > mol selection "residue 1 to 20"
vmd > mol addrep 0
vmd > mol list 0
Status of molecule 1ubq:
1ubq  Atoms:660  Frames (C): 1(0)  Status:ADfT
Atom representations: 2
0:  on, 660 atoms selected.
  Coloring method: Name
   Representation: Lines
        Selection: all
1:  on, 155 atoms selected.
  Coloring method: Name
   Representation: Lines
        Selection: residue 1 to 20
```

mol 0 Rep 1 のスタイルを NewCartoon (リボン図) に変更

```
vmd > mol modstyle 1 0 NewCartoon
Info) In any publication of scientific results based in part or
Info) completely on the use of the program STRIDE, please reference:
Info)  Frishman,D & Argos,P. (1995) Knowledge-based secondary structure
Info)  assignment. Proteins: structure, function and genetics, 23, 566-579.
```

mol 0 Rep 1 の色を Chain に変更

```
vmd > mol modcolor 1 0 chain
```

mol 0 rep 1 の色を ColorID: 10 (cyan) に変更

```
vmd > mol modcolor 1 0 ColorID 10
```

## アニメーション

Trajectory 表示中に使えるコマンド

```
animate forward		# 再生
animate reverse		# 逆再生
animate pause		# 停止
animate next		# 1コマ進む
animate prev		# 1コマ戻る
animate skip 10		# スキップ幅を指定
animate speed 0.5	# 再生速度を指定 (0〜1)
animate goto start	# 最初に戻る
animate goto end 	# 最後に行く
animate goto 20		# 20フレーム目へ移動
```

## 現在表示中の画面を保存

```
vmd > render snapshot snap.tga
```

tga とか rgb とかいった謎形式でしか保存できないが convert (Imagemagick) を使えば png や jpeg などの社会的な形式に変更できる。

## 原子団を選択

molID: 0 に含まれている全原子を選択

```
vmd > atomselect 0 "all"
```

選択結果を変数 `$sel` に格納

```
vmd > set sel [atomselect 0 "all"]
```

`$sel` に含まれる原子数を表示

```
vmd > $sel num
660
```

他にも色々な情報を取得できる

```
$sel get resid		# 原子団の残基番号
$sel get resname	# 原子団の残基名
$sel get x			# 原子団の x 座標
$sel moveby {10 0 0}	# 原子団を並進移動
$sel move [transaxis x 40 deg]		# 原子団をx軸中心に40度回転
```

他にも `getbonds` で原子結合の一覧を表示したり `moveto` で動かしたり出来るが詳しくは [公式ドキュメント](http://www.ks.uiuc.edu/Research/vmd/current/ug/node122.html) を参照。

## 参考

- [VMD User's Guide](http://www.ks.uiuc.edu/Research/vmd/current/ug/ug.html)

