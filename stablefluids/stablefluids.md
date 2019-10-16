1999 Jos Stam

# Stable Fluids 訳とメモ書き

## Abstract

この時代の前まで、不安定なschemeで演算してたせいでリアルタイム流体シミュレーションは厳しかったらしい。   
この行ちょっとよくわからん。テクスチャに描き出すってことか？

>  We have used our model in conjuction with advecting solid textures to create many fluid-like animations interactively in two- and three-dimensions

## Introduction

いろんな業界で流体シミュにはナビエストークス方程式が良いと言われているらしい。   
CGでは流体の形と振る舞いが特に重要。(物理学的な正しさは割と二の次)   
**Fluid solvers** によってリアルタイム流体エフェクトを提供する。   

 #### early works

- パーティクルシステムによる葉などの簡単なジオメトリの流動アニメーション
- 高速な乱気流シミュ -> 時間と空間で離散的に実装されたためテクスチャマッピングに合致した
- ナビエストークスの最初の実装は２次元でされたが、unstableで回転運動もできなかった
- 最近(1999当時)になって３次元でも実装されたが、large time step で unstableになる

#### stable, unstable

流体シミュでいう`stable`とは、ソルバーが返す結果の値がどんな条件でも異常値にならないことらしい。     
  