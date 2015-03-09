---
layout: post
title: GROMACS 使ってみる
date: 2015-03-09
tag: gromacs
---

本記事は [Lemkul 氏の GROMACS Tutorial](http://www.bevanlab.biochem.vt.edu/Pages/Personal/justin/gmx-tutorials/lysozyme/index.html) の [Tutorial 1: Lysozyme in Water](http://www.bevanlab.biochem.vt.edu/Pages/Personal/justin/gmx-tutorials/lysozyme/index.html) に基づく。

GROMACS のバージョンは 5.0.4, OS は Ubuntu 14.04 を用いた。

# 基礎論

そもそも MD (分子動力学) とは、単純に言えば **やたら複雑な微分方程式を積分する** だけの作業である。
従って入力値として必要なものは

* どんな微分方程式なのか？ (topology)
* どんな初期値なのか？ (原子座標)

この2つである。

従って、GROMACS における分子計算は、以下の手順で行われる。

* 原子座標などの .gro ファイルと topoloty である .top ファイルを作成する。
* これらのファイルに必要な加工 (溶媒の追加など) を行う。
* 「積分」を実行する。

以下、それぞれについて説明する。

# 構造と topology ファイルの生成

まず適当な作業用ディレクトリを作り [Protein Data Bank](http://pdb.org/) から Lysozyme (PDB code: `1AKI`) をダウンロードする。

![image](/images/gmx-1AKI.png)

次に、以下のコマンドを実行する。

```
$ gmx pdb2gmx -f 1AKI.pdb -o 1AKI_processed.gro -water spce
```

これは `1AKI.pdb` という PDB ファイルから topology などのファイルを出力するというコマンドであり `-water spce` は水のモデルとして SPC/E を使うという意味である。

実行すると Force Field (力場) を選べと言われる。力場とは「どの結合に対し、どんな力を適用するか」といった情報の集合である。色々な研究者が目的に応じて様々な力場を提案しているのだが、今回はどれでもいいので、`15` (OPLS-AA/L all-atom force field (2001 aminoacid dihedrals) を選択。すると3つのファイルが生成される。

```
$ ls
1AKI.pdb	1AKI_processed.gro	posre.itp	topol.top
```

全てテキストファイルであるため、開いてみればなんとなく意味は分かる。たとえば `1AKI_processed.gro` は 1AKI に含まれる原子の座標情報であり、冒頭部分を読むと


```
$ head 1AKI_processed.gro
GROningen MAchine for Chemical Simulation
 1960
    1LYS      N    1   3.536   2.234  -1.198
    1LYS     H1    2   3.612   2.288  -1.236
    1LYS     H2    3   3.470   2.214  -1.270
    1LYS     H3    4   3.492   2.286  -1.125
    1LYS     CA    5   3.589   2.107  -1.143
    1LYS     HA    6   3.633   2.055  -1.216
    1LYS     CB    7   3.687   2.144  -1.031
    1LYS    HB1    8   3.763   2.195  -1.070
```

となる。1960 個の原子があり、その原子名と番号と座標がつらつら書いてあるだけのファイルだ。この時点ではまだ溶媒水は含まれていない。また最後の行には

```
$ tail 1AKI_processed.gro -n 1
   3.81674   4.23486   3.45459
```

とある。これは box のサイズであり、現在はこのタンパク質がギリギリ収まるサイズになっている。
また `posre.itp` とは position restriction 即ち拘束条件で、今回は特にこれといった拘束は用いないが、脂質膜の計算などを行う時に活躍する。 `topol.top` が topology で、それぞれの原子にどのような電荷があるか、どの原子とどの原子が結合しているか、そこにどのような力が適用されるか、といった情報がつらつらと書いてある。

次に、この simulation box を適切に拡張する。

```
$ gmx editconf -f 1AKI_processed.gro -o 1AKI_newbox.gro -c -d 1.0 -bt cubic
```

これで `1AKI_newbox.gro` というファイルが新たに生成される。`1AKI_processed.gro` の box サイズを拡張し `7.01008` の立方体とし、さらにタンパク質がその中央に来るように平行移動している。
オプションの `-c` は *center*, `-d 1.0` はタンパク質を box の面から最低 1.0 以上離すという意味である。 `-bt cubic` は box type を立方体にするという意味。

# 溶媒の追加と中和

続いて、ここに溶媒水を入れる。

```
$ gmx solvate -cp 1AKI_newbox.gro -cs spc216.gro -o 1AKI_solv.gro -p topol.top
```

これで溶媒水の入った構造ファイル `1AKI_solv.gro` が生成されるとともに、その情報が `topol.top` に追記される。
オプションの `-cp` は configure of protein, `-cs` は configure of solvent である。 `spc216.gro` は溶媒水の構造ファイルで、普通にインストールしたならば `/usr/local/gromacs/share/gromacs/top` に入っている。

さて、もともと Lysozyme (1AKI) は塩基性アミノ酸を多く含むため、この時点で系全体は +8 に帯電している。そこで、いくつかの水分子を陰イオンに置換し、系全体の電荷をゼロにする。

まず `ions.mdp` というファイルを作成し、内容を以下のようにする。

```
; ions.mdp - used as input into grompp to generate ions.tpr
; Parameters describing what to do, when to stop and what to save
integrator	= steep		; Algorithm (steep = steepest descent minimization)
emtol		= 1000.0  	; Stop minimization when the maximum force < 1000.0 kJ/mol/nm
emstep      = 0.01      ; Energy step size
nsteps		= 50000	  	; Maximum number of (minimization) steps to perform

; Parameters describing how to find the neighbors of each atom and how to calculate the interactions
nstlist		    = 1		    ; Frequency to update the neighbor list and long range forces
cutoff-scheme   = Verlet
ns_type		    = grid		; Method to determine neighbor list (simple, grid)
coulombtype	    = PME		; Treatment of long range electrostatic interactions
rcoulomb	    = 1.0		; Short-range electrostatic cut-off
rvdw		    = 1.0		; Short-range Van der Waals cut-off
pbc		        = xyz 		; Periodic Boundary Conditions (yes/no)
```

次に、以下のコマンドを実行する。

```
$ gmx grompp -f ions.mdp -c 1AKI_solv.gro -p topol.top -o ions.tpr
```

これで `ions.tpr` というバイナリファイルが作成される。次に以下のコマンドでイオンを添加する。

```
$ gmx genion -s ions.tpr -o 1AKI_solv_ions.gro -p topol.top -pname NA -nname CL -nn 8
```

*Select a continuous group of solvent molecules* という指示が出るので、13 (SOL) を選択。これで 32124 個ある溶媒分子の8個が Cl ions に置換され、系全体が中和されるたファイルが `1AKI_solv_ions.gro` として出力され、また `topol.top` も一部が書き換わる。

# エネルギー最小化

これで構造自体は完成したが、このままでは分子のどこかに衝突が起きていて、MD を実行すると途端に分子が崩壊してしまう可能性がある。そこで MD 実行前に、まずエネルギー最小化を行い、歪みを除いておく必要がある。

まず `minim.mdp` というファイルを作成。最小化の詳細な手順について記述する。

```
; minim.mdp - used as input into grompp to generate em.tpr
integrator	= steep		; Algorithm (steep = steepest descent minimization)
emtol		= 1000.0  	; Stop minimization when the maximum force < 1000.0 kJ/mol/nm
emstep      = 0.01      ; Energy step size
nsteps		= 50000	  	; Maximum number of (minimization) steps to perform

; Parameters describing how to find the neighbors of each atom and how to calculate the interactions
nstlist		    = 1		    ; Frequency to update the neighbor list and long range forces
cutoff-scheme   = Verlet
ns_type		    = grid		; Method to determine neighbor list (simple, grid)
coulombtype	    = PME		; Treatment of long range electrostatic interactions
rcoulomb	    = 1.0		; Short-range electrostatic cut-off
rvdw		    = 1.0		; Short-range Van der Waals cut-off
pbc		        = xyz 		; Periodic Boundary Conditions (yes/no)
```

これを用いて `em.tpr` というバイナリファイルを作成。

```
$ gmx grompp -f minim.mdp -c 1AKI_solv_ions.gro -p topol.top -o em.tpr
```

標準出力に `This run will generate roughly 3 Mb of data` と、出力ファイルが大雑把にどのくらいになるのかを予告してくれる。

それでは最小化を実行。

```
$ gmx mdrun -v -deffnm em
```

`-v` は「途中経過を表示する」という意味で `-deffnm` は「出力ファイルの名前はデフォルトに従う」という意味。
どちらも計算の本質とは特に関係ない。

計算は数秒で終了し、これにより4つのファイルが生成される。

* em.log : 計算内容の要約。
* em.edr : エネルギーファイル。計算途中でエネルギーがどのように変化したかを記録している。
* em.trr : Trajectory ファイル。計算途中で原子座標がどのように変化したかをイロクしている。
* em.gro : 最終的に得られた、エネルギー最安定化された構造。

エネルギーの変化については `em.edr` に格納されているが、これはバイナリファイルであるため、読める形にするには

```
$ gmx energy -f em.edr -o potential.xvg
```

というコマンドを実行する。「どの部分のエネルギーを出力しますか」と聞かれるので `10 0` と入力すると potential が得られる。 (Bond や Angle など個別のエネルギーを出力することも出来る。)

`potential.xvg` はテキストファイルなので、適当に捌けばプロットを表示するｋとが出来る。

![image](/images/gmx-energy-minimize.png)

# 平衡化

無事最小化されたようなので、この構造を初期値として MD を実行する。といっても、最小化との違いは .mdp ファイルの中身だけである。

今回実行するのは、体積・温度一定条件下における平衡化計算である。いわゆる NVT ensemble。
まず `nvt.mdp` というファイルを作成する。

```
title		= OPLS Lysozyme NVT equilibration 
define		= -DPOSRES	; position restrain the protein
; Run parameters
integrator	= md		; leap-frog integrator
nsteps		= 50000		; 2 * 50000 = 100 ps
dt		    = 0.002		; 2 fs
; Output control
nstxout		= 500		; save coordinates every 1.0 ps
nstvout		= 500		; save velocities every 1.0 ps
nstenergy	= 500		; save energies every 1.0 ps
nstlog		= 500		; update log file every 1.0 ps
; Bond parameters
continuation	        = no		; first dynamics run
constraint_algorithm    = lincs	    ; holonomic constraints 
constraints	            = all-bonds	; all bonds (even heavy atom-H bonds) constrained
lincs_iter	            = 1		    ; accuracy of LINCS
lincs_order	            = 4		    ; also related to accuracy
; Neighborsearching
cutoff-scheme   = Verlet
ns_type		    = grid		; search neighboring grid cells
nstlist		    = 10		; 20 fs, largely irrelevant with Verlet
rcoulomb	    = 1.0		; short-range electrostatic cutoff (in nm)
rvdw		    = 1.0		; short-range van der Waals cutoff (in nm)
; Electrostatics
coulombtype	    = PME	; Particle Mesh Ewald for long-range electrostatics
pme_order	    = 4		; cubic interpolation
fourierspacing	= 0.16	; grid spacing for FFT
; Temperature coupling is on
tcoupl		= V-rescale	            ; modified Berendsen thermostat
tc-grps		= Protein Non-Protein	; two coupling groups - more accurate
tau_t		= 0.1	  0.1           ; time constant, in ps
ref_t		= 300 	  300           ; reference temperature, one for each group, in K
; Pressure coupling is off
pcoupl		= no 		; no pressure coupling in NVT
; Periodic boundary conditions
pbc		= xyz		    ; 3-D PBC
; Dispersion correction
DispCorr	= EnerPres	; account for cut-off vdW scheme
; Velocity generation
gen_vel		= yes		; assign velocities from Maxwell distribution
gen_temp	= 300		; temperature for Maxwell distribution
gen_seed	= -1		; generate a random seed
```

意味はなんとなく読めばわかると思う。ポイントは time step が 2 fs である事、それを 50,000 steps つまり 100 ps 実行すること、温度が 300 K である事など。
ここからエネルギー最小化の時と同様に `.tpr` ファイルを作成し、計算を実行する。

```
$ gmx grompp -f nvt.mdp -c em.gro -p topol.top -o nvt.tpr
$ gmx mdrun -deffnm nvt
```

計算は5分ほどで終了。以下のような要約が出る。

```
NOTE: The GPU has >20% more load than the CPU. This imbalance causes
      performance loss, consider using a shorter cut-off and a finer PME grid.

               Core t (s)   Wall t (s)        (%)
       Time:     1904.055      239.372      795.4
                 (ns/day)    (hour/ns)
Performance:       36.095        0.665
```

NOTE によると「GPU の使用率が CPU より2割以上多い」つまりGPU律速になっている事が分かる。GPU の役割は長距離力の計算であるため、この計算を行う範囲を狭める (cut-off を短くする) と良いよ、などのアドバイスが出る。

元のチュートリアルではこのあと NPT 平衡化を 100 ps にわたって実行し、その上で 1000 ps の production MD を実行するが、そのへんは mdp ファイルが違うだけで一緒なので各自参照のこと。

# 結果をVMDで表示する

周期境界条件があるので、前処理として `trjconv` をしておく必要がある。

```
$ trjconv -s nvt.tpr -f nvt.trr -o nvt_noPBC.xtc -pbc mol -ur compact
```

```
$ vmd -gro nvt.gro -xtc nvt_noPBC.xtc
```

![image](/images/gmx-lysozyme-vmd.png)

