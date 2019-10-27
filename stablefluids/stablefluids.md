1999 Jos Stam

# Stable Fluids 訳とメモ書き

## Abstract

この時代の前まで、不安定なschemeで演算してたせいでリアルタイム流体シミュレーションは厳しかったらしい。   
この行ちょっとよくわからん。テクスチャに描き出すってことか？

>  We have used our model in conjuction with advecting solid textures to create many fluid-like animations interactively in two- and three-dimensions

## 1章 Introduction

いろんな業界で流体シミュにはナビエストークス方程式が良いと言われているらしい。   
CGでは流体の形と振る舞いが特に重要。(物理学的な正しさは割と二の次)   
**Fluid solvers** によってリアルタイム流体エフェクトを提供する。   

### early works

- パーティクルシステムによる葉などの簡単なジオメトリの流動アニメーション
- 高速な乱気流シミュ -> 時間と空間で離散的に実装されたためテクスチャマッピングに合致した
- ナビエストークスの最初の実装は２次元でされたが、unstableで回転運動もできなかった
- 最近(1999当時)になって３次元でも実装されたが、large time step で unstableになる

### stable, unstable

流体シミュでいう`stable`とは、ソルバーが返す結果の値がどんな条件でも異常値にならないことらしい。     

### Foster&Metaxa の旧来の手法から紹介

ナビエストークス方程式を解くためのラグランジアンを使って彼らの手法を解説するんだって。   
急速に弱まったりしやすく、雑すぎて実用には耐えないがcomputer-interactionならそんなもんだし大丈夫やろ的なこと書いてある。   
流れとモーションの密度？をソルバーによって解決する。微細な機体の動きを表現するのに密度が大事らしい。   
このペーパーの流れ

- 2章でナビエストークスからStableFluidsを導出(専門用語など多め)
- ↑難しいから単なるアプリ実装者は2を飛ばして3行っていいらしい
- 3章で実装のしかた
- 4章で優れた点解説
- 5章で結論
- コンピュータエンジニアリングな人におすすめの本らしい Computational Fluid Dynamics: An Introduction for Engineers. 

## 2章 Stable Navier-Stokes

だいたい紙に書いてる   

#### 境界条件





## 3章 Our Solver

### Setup

流体の動きと、流体に混じった物質(煙とか塵とか)の伝播両方がこの実装で再現できるらしい。
`NDIM=座標空間の次元`とすると、

- 原点の位置ベクトル`O[NDIM]`
- ある軸の長さを`L[NDIM]`に格納
-  ある軸上のセル数を`N[NDIM]`に格納
-   ある軸上のボクセルのサイズ`D[i]=L[i]/N[i]`

各ボクセルの中心に`velocity field`がある。(旧来研究はボクセル境界に`velocity field`を置いていた。)   
velocityの保存のために2つののグリッド`U0[NDIM]`, `U1[NDIM]`を用意してダブルバッファリング的に利用する。   
更にそのベロシティによって移動した物質の状態を保存するための2つのグリッド`S0[NDIM]`, `S1[NDIM]`も用意するらしい。   
`single time step`, つまりシュミュレーション中の1フレームの経過秒数を`dt`で表す。   
流体の物理的特性(`physical properties`)は`viscosity`, 粘性のみで、`visc`関数で表す。   
物質の特性は拡散係数?(`diffusion constant`) `kS` と散逸率(`dissipation rate`) `aS`で表す。   
**gridの境界条件っていうのを`periodic`か`fixed`に決めなきゃいけない?? ようわからん(p5の左上)**   
流体は外力によって動く

### Simulator

ベロシティフィールドを`Vstep`関数で、スカラーフィールド(要は位置など)を`Sstep`関数で毎フレーム更新する。   

#### ベロシティソルバーの処理ステップ `Vstep()`

- 外力の値をフィールドに加える `addForce()`
- ベロシティフィールドが移流する `transport()`
- ベロシティフィールドが(他グリッドへ？)流体の粘性と摩擦によって拡散する `diffuse()`
- ベロシティを求め終わって、実際に力を加える処理をする `project()`

#### スカラーフィールドソルバーの処理ステップ `Sstep()`

ベロシティソルバーと同じような感じと書いてある。  
3ステップ目と4ステップ目がなんのことだかイマイチわからん。同じことしてねえか？

- 外力を加える `addForce()`
- 前に求めたベロシティによって移流する?(移動する?) `transport()`
- 他グリッドへ拡散する `diffuse()`
- 散逸する?? `dissipate()`

##### `Sstep::transport()`の処理ステップ

全てのセル(`cell` == `S1[i, j, k]`)に対して:  `traceParticle()`と`cell = lerp()`をやる   

##### `Sstep::transport::traceParticle()`の処理ステップ

書こうと思ったけどやめる。   


↑実装の話は `REAL-TIME FLUID DYNAMICS FOR GAMES(GDC 2003)`の方を読んでみる。

