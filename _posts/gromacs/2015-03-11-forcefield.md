---
layout: post
title: Forcefield 力場について
date: 2015-03-11
categories: gromacs
permalink: /:categories/:title
---

GROMACS の力場についてはマニュアルの4章に詳しいが、ここでは概略と力場ファイルの読み方だけ述べる。

# 基本

そもそも力場とは

* potential function
* parameters

この2つの組み合わせを意味する。potential function は GROMACS に組み込まれている力場を番号で指定し、parameter は数値で指定する。

## GROMACS で用いられる potential function

non-bonded, bonded, restriction の3つに大別される。各ポテンシャルに複数の関数形が存在し、力場ファイル内で選択するものが多い。（bond-stretch ならば調和振動・Morseなど）

|カテゴリ|ポテンシャル|関数|備考|
|:--:|:--:|:--:|:--:|
|Non-bonded|vdW|Lennard-Jones|6乗の引力と12乗の斥力|
||vdW|Buckingham|6乗の引力と指数関数の斥力|
||Coulomb|距離に反比例|PMEなど特殊な計算法を要する|
|Bonded|stretching|調和振動||
||stretching|Morse|1-exp(⊿r)|
||stretching|cubic|⊿rの2乗と3乗の和|
||angle|cosine-based||
||angle|Urey-Bradley||
||dihedral|||
||cross-section|bond-bond| bond-angle 等|
|Restratint|position|調和振動|空間中の1点に対する調和振動子|

# 力場ファイル

GROMACS のインストール時に特に指定しなければ
`/usr/local/gromacs/share/gromacs/top`
に力場がある。力場は複数のファイルが1個のディレクトリにまとめられており、それぞれ `amber03.ff`, `charmm27.ff`, `gromos54a7.ff` といった名前が付けられている。

ディレクトリの中はだいたい以下のようなファイルがある。

* forcefield.itp
* ffnonbonded.itp
* ffbonded.itp
* 各分子の力場

この forcefield.itp が（恐らく）最初に読まれる力場であり、内容は以下のようになっている。(gromos53a6.ffの例)

```
$ cat forcefield.itp
#define _FF_GROMOS96
#define _FF_GROMOS53A6

[ defaults ]
; nbfunc comb-rule gen-pairs fudgeLJ fudgeQQ
  1    1   no    1.0 1.0

#include "ffnonbonded.itp"
#include "ffbonded.itp"
```

ここで nbfunc は non-bonding function であり、1がLennard-Jones、2がBuckinghamを表す。それ以外の項目は知らない。

最後の2行で `ffnonbonded.itp`, `ffbonded.itp` という2つのファイルを読み込んでいるが、これらはそれぞれ non-bonding および bonding の力場が記述されている。
たとえば `ffnonbonding.itp` には

```
[ atomtypes ]
;name  at.num   mass      charge  ptype       c6           c12
    O    8   0.000      0.000     A  0.0022619536       1e-06
   OM    8   0.000      0.000     A  0.0022619536  7.4149321e-07
   OA    8   0.000      0.000     A  0.0022619536  1.505529e-06
   OE    8   0.000      0.000     A  0.0022619536    1.21e-06
   OW    8   0.000      0.000     A  0.0026173456  2.634129e-06
    N    7   0.000      0.000     A  0.0024364096  2.319529e-06
   NT    7   0.000      0.000     A  0.0024364096  5.0625e-06
   NL    7   0.000      0.000     A  0.0024364096  2.319529e-06
   NR    7   0.000      0.000     A  0.0024364096  3.389281e-06
(...)
```

となっており、各原子種の原子量や電荷、LJ potential の係数が書かれている。異なる2原子間のc6, c12の値は、gromos系力場では[ nonbond_params ]という項目で個別に設定されているが、amber系力場ではされておらず、各原子の力場の幾何平均、もしくは Lorentz-Berthelot rules により算出する。

さらに `ffbonding.itp` では各分子の力場が include されており、たとえば水の力場である tip3p.itp には

```
[ moleculetype ]
; molname  nrexcl
SOL    2

[ atoms ]
; id at type res nr  residu name at name   cg nr charge
1       OWT3    1       SOL              OW             1       -0.834
2       HW      1       SOL             HW1             1        0.417
3       HW      1       SOL             HW2             1        0.417
```

とある。

# 力場の使用・編集

GROMACS では力場は topology ファイルで指定されている。`pdb2gmx` で top ファイルを作成した場合、その冒頭部にたとえば

```
#include "gromos54a7.ff/forcefield.itp"
```

このように力場が include されている。独自の力場を追加したい場合、同様にここに 

```
#include "myff.ff"
```

のように書き足すことになる。
