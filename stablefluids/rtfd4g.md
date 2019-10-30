2003 Jos Stam

# Real-Time Fluid Dynamics for Games訳とメモ書き

## Abstract, Introduction

[ここ](https://shikihuiku.wordpress.com/2013/05/02/real-time-fluid-dynamics-for-gamesを読んでみる/)と[ここ]( [https://shikihuiku.wordpress.com/2013/05/07/real-time-fluid-dynamics-for-games%e3%82%92%e8%aa%ad%e3%82%93%e3%81%a7%e3%81%bf%e3%82%8b2/](https://shikihuiku.wordpress.com/2013/05/07/real-time-fluid-dynamics-for-gamesを読んでみる2/) )に日本語訳があるので読みながらメモする。

#### この章の大事なところ

- `velocity field` はある空間に充満した流体を表す、「ある時間におけるある場所の速度の配列」
- ナビエストークスは、流体の時間的進展("動き"や"速度変化"と言い換えてもいいかもしれない)を表す式。
- densityは、ある空間内の粒子の充填率を`[0.0, 1.0]`で表す。

## A Fluid in a Box

![fluidinbox](C:\Users\albus\Documents\thesisnotes\stablefluids\fluidinbox.png)

**訳**) 簡単のため、厳密な式などは利用せず2次元の例として示しますが、3次元への拡張も自然に可能です。空間をグリッド分割し、各セルの中心点で速さなどを処理します。また、処理領域の1マス外側に余分なマスを用意して処理領域を囲み、境界として利用します。速度と密度のfieldは、サイズが`(N+2)^DIMENSION`の配列:   
`u`, `u_prev`,  `v`, `v_prev`, `dens`, `dens_prev`   
として保存します。

## Moving Densities

ここも日本語訳を見る。

#### この章の大事なところ

- densityの計算ステップはだいたいこんな感じ。   
  `N`はフィールドの幅、`x` `x0`はdensity field、`u` `v`はverocity field、`diff`は拡散定数、`dt`は単位時間。

  ```c++
  void dens_step(int N, float* x, float* x0, float* u, float* v, float diff, float dt){
      add_source ( N, x, x0, dt );
      
      SWAP ( x0, x );   
      diffuse ( N, 0, x, x0, diff, dt );
      
      SWAP ( x0, x );
      advect ( N, 0, x, x0, u, v, dt );
  }
  ```

  

密度に関するナビスト式の右辺に3要素を与えることによって、時間経過単位`dt`ごとの密度の変化量を求める。それらは以下の通り。

- 1番目の項 : 密度はベロシティフィールドに従って変化する。`s`は"source", `x`はdensity field

  ```C++
  void add_source ( int N, float * x, float * s, float dt ){
  	for ( int i=0, size=(N+2)*(N+2); i<size ; i++ ){
          x[i] += dt*s[i];
      }
  }
  ```

- 2番目の項 : 密度は一定の確率`diff`(Fig.1のkみたいな見た目の定数)で周囲へ拡散する。

  `diff > 0`の時に密度は周囲のマスへ拡散する。
  周囲からの流れ込み&周囲への流れだしは、結果的に中央のマス(今注目しているマス)と周囲4マスの密度の差分で表すことができる。

  ```C++
  void diffuse_bad ( int N, int b, float * x, float * x0, float diff, float dt ){
      float a=dt*diff*N*N;
      for ( int i=1 ; i<=N ; i++ ) {
          for ( int j=1 ; j<=N ; j++ ) {
              x[IX(i,j)] = x0[IX(i,j)]
  			           + a*(  x0[IX(i-1,j)] + x0[IX(i+1,j) + x0[IX(i,j-1)]
  			                 +x0[IX(i,j+1)] - 4*x0[IX(i,j)]);
          }
      }
      set_bnd ( N, b, x );
  }
  ```

  上の手法は拡散率`a`が大きい時に**シミュが破綻してしまうので、良くない**。

  **もう一つ、線形システムを利用した実装がある。**が、これは今回においては"過剰"ゆえ採用しない。

  最終的には、**Gauss-Seidel法(下のコード)**で実装するとよさそう。

  ```C++
  void diffuse ( int N, int b, float * x, float * x0, float diff, float dt ){
      float a=dt*diff*N*N; // なんでNを2回もかけてんねん
      for ( int k=0 ; k<20 ; k++ ) {
          for ( int i=1 ; i<=N ; i++ ) {
              for ( int j=1 ; j<=N ; j++ ) {
                  x[IX(i,j)] = ( x0[IX(i,j)]+
                                 a*(x[IX(i-1,j)]+ // 1つ左のマス
                                    x[IX(i+1,j)]+ // 1つ右のマス
                                    x[IX(i,j-1)]+ // 1つ下のマス
                                    x[IX(i,j+1)]) // 1つ上のマス
                               )/(1+4*a);
              }
          }
          set_bnd ( N, b, x );
      }
  }
  ```

  

- 3番目の項 : 外的な力で密度が変化する(上昇のみっぽい?)。

  ベロシティ依存なので普通にこの項を解くのは大変であり、今注目しているマスにパーティクルがあると考え、それが前フレーム(`dt`時間だけ前)にどの位置(マス)にいたかを求めて(トレースして)利用する。

  ```C++
  void advect (int N, int b, float * d, float * d0, float * u, float * v, float dt){
      float dt0 = dt*N; // なんでNかけてんねん
      for ( int i=1 ; i<=N ; i++ ) {
          for ( int j=1 ; j<=N ; j++ ) {
              float x = i-dt0*u[IX(i,j)]; // 現在の座標i,jから(経過時間*速度)を
              float y = j-dt0*v[IX(i,j)]; // 引いて前フレームの場所を求める
              if (x<0.5){ // 境界を超えるなら引き戻す
                  x=0.5;
              }
              if (x>N+0.5){ // 上と同じ
                  x=N+ 0.5;
              }
              int i0=(int)x; // 前フレームの座標のidx
              int i1=i0+1;   // 前フレームの座標の右隣のidx
              if (y<0.5){
                  y=0.5;
              }
              if (y>N+0.5){
                  y=N+ 0.5;
              }
              int j0=(int)y;   // 前フレームの座標のidx
              int j1=j0+1;     // 前フレームの座標の一個上(か下かわからんけども)のidx
              float s1 = x-i0; // 前フレームの位置と現在の座標の距離(間にあるマスの数)
              float s0 = 1-s1; // ﾅﾆｺﾚ?
              float t1 = y-j0; // 前フレームの位置と現在の座標の距離(間にあるマスの数)
              float t0 = 1-t1; // ﾅﾆｺﾚ?
              d[IX(i,j)] = s0*(t0*d0[IX(i0,j0)] + t1*d0[IX(i0,j1)])+
                           s1*(t0*d0[IX(i1,j0)] + t1*d0[IX(i1,j1)]);
          }
      }
      set_bnd ( N, b, d );
  }
  ```

  

- この実装では↑のすべての要素を3番目から逆順に、時間`dt`ごとに更新する。   
  

## Evolving Velocities

これも日本語訳から。

#### この章の大事なところ

velocity の計算ステップはだいたいこんな感じ。   
`N`はフィールドの幅、`x` `x0`はdensity field、`u` `v` `u0` `v0`はverocity field、`visc`は粘性率? `dt`は単位時間。

```c++
void vel_step (int N, float* u, float* v, float* u0, float* v0, float visc, float dt){
    add_source ( N, u, u0, dt );
    add_source ( N, v, v0, dt );
    
    SWAP ( u0, u );
    diffuse ( N, 1, u, u0, visc, dt );
    
    SWAP ( v0, v );
    diffuse ( N, 2, v, v0, visc, dt );
    
    project ( N, u, v, u0, v0 );
    SWAP ( u0, u ); SWAP ( v0, v );
    
    advect ( N, 1, u, u0, u0, v0, dt );
    advect ( N, 2, v, v0, u0, v0, dt );
    
    project ( N, u, v, u0, v0 );
}
```



- 密度と同様、速度に関するナビスト式の3つの項を解く。`diffuse()` `add_source()` `advect()`は全く同じものを流用できる。

- 新たに登場した`project()`は流体の質量保存に関する重要な関数らしい。渦がどうとか書いてあるのでたぶん例の**Vorticity confinement**じゃないか？下記は実装例。

  ```C++
  void project ( int N, float * u, float * v, float * p, float * div ){
      float h = 1.0/N;
      for ( i=1 ; i<=N ; i++ ) {
          for ( j=1 ; j<=N ; j++ ) {
              div[IX(i,j)] = -0.5*h*(u[IX(i+1,j)]-u[IX(i-1,j)]+
              v[IX(i,j+1)]-v[IX(i,j-1)]);
              p[IX(i,j)] = 0;
          }
      }
      set_bnd ( N, 0, div ); set_bnd ( N, 0, p );
      for ( int k=0 ; k<20 ; k++ ) {
          for ( int i=1 ; i<=N ; i++ ) {
              for ( int j=1 ; j<=N ; j++ ) {
                  p[IX(i,j)] = (div[IX(i,j)]+p[IX(i-1,j)]+p[IX(i+1,j)]+
                   p[IX(i,j-1)]+p[IX(i,j+1)])/4;
              }
          }
          set_bnd ( N, 0, p );
      }
      for ( int i=1 ; i<=N ; i++ ) {
          for ( int j=1 ; j<=N ; j++ ) {
              u[IX(i,j)] -= 0.5*(p[IX(i+1,j)]-p[IX(i-1,j)])/h;
              v[IX(i,j)] -= 0.5*(p[IX(i,j+1)]-p[IX(i,j-1)])/h;
          }
      }
      set_bnd ( N, 1, u );
      set_bnd ( N, 2, v );
  }
  ```

- 最終的にmainはこんな感じに書ける。

  ```C++
  while ( simulating ){
      get_from_UI ( dens_prev, u_prev, v_prev );
      vel_step ( N, u, v, u_prev, v_prev, visc, dt );
      dens_step ( N, dens, dens_prev, u, v, diff, dt );
      draw_dens ( N, dens );
  }
  ```

  

## わからんとこリスト

- Gauss-Seidel法
- ナビエストークス方程式のナブラ項の意味
- 数式から実装におとしこむ全体的なアイデアが分からない