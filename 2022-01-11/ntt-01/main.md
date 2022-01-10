---
title: "数論変換(NTT)を用いた多項式乗算① 〜離散フーリエ変換と多項式乗算〜"
tags:
  - 数学
  - Rust
---

# はじめに

この記事は数論変換(NTT)を用いた畳み込みについて1から丁寧に解説することを目標としています。この記事を書こうと思ったきっかけは学校で｢円周率を1000桁計算する｣というレポートが出たからです。いわゆる多倍長演算のレポートなのですが、乗算の速度をなんとかして上げようと思い、高速フーリエ変換や数論変換について学びました。結局多倍長演算にはNTTは使わなかったのですが、学んだ内容をどこかに残しておきたかったのでここに書きます。僕自身、数学はあまり得意では無いのでもしかしたら間違っているかもしれませんが、そのときはTwitterやGitHubで教えてもらえると嬉しいです。

第1回では離散フーリエ変換と多項式乗算の関係について解説します。

# 多項式乗算

多項式乗算とはその名の通り、多項式同士の乗算です。例えば2つの多項式$f(x) = 1 + 2x$と$g(x) = 3 + x + 4x^2$が与えられたとき、この2式の乗算結果は$f(x)g(x) = 3 + 7x + 6x^2 + 8x^3$となります。プログラム上で表現する場合はそれぞれの項を配列に格納することで表現できます。入力の配列の長さをそれぞれ$n, m$とすると、出力の配列の長さは$n + m - 1$となります。2重forを用いた非常に単純な実装を示します。

```rust
fn mul(f: &Vec<i64>, g: &Vec<i64>) -> Vec<i64> {
    let mut ans = vec![0; f.len() + g.len() - 1];
    
    for (i, ff) in f.iter().enumerate() {
        for (j, gg) in g.iter().enumerate() {
            ans[i + j] += ff * gg;
        }
    }
    
    ans
}
```

この2重forを用いた実装は単純ですが計算量は$O(nm)$となり、入力が大きい場合は計算に時間がかかります。どうにかしてこれを高速化できないでしょうか？ここで使用するのが高速フーリエ変換です。

# 高速フーリエ変換

高速フーリエ変換(FFT)は離散フーリエ変換を高速で行うことができるアルゴリズムです。計算量は$O(n \log n)$となり$O(nm)$に比べて非常に高速です。FFTをうまく使用することで多項式乗算を行うことができます。

# 離散フーリエ変換

FFTについて解説する前にまずは離散フーリエ変換(DFT)について学びましょう。[Wikipedia](https://ja.wikipedia.org/wiki/%E9%9B%A2%E6%95%A3%E3%83%95%E3%83%BC%E3%83%AA%E3%82%A8%E5%A4%89%E6%8F%9B)によると、

> 離散フーリエ変換とは、複素関数$f(x)$を複素関数$\hat f(t)$に写す写像であって…(略)

だそうです(?)。定義は以下の通りです。
$$
\begin{align*}
\hat{f}(t) &= \sum^{N - 1}_{i = 0} f(\zeta^i_N) t^i \\
&= f(\zeta_N^0)t^0 + f(\zeta_N^1)t^1 + f(\zeta_N^2)t^2 + \cdots + f(\zeta_N^{N-2})t^{N-2} + f(\zeta_N^{N-1})t^{N-1}
\end{align*}
$$
見慣れない$\zeta_N$というものが出てきました。これは｢1の原始$N$乗根｣と呼ばれるもので、$N$乗して初めて1になる数のことです。複素数の範囲で表すと以下のようになります。
$$
\zeta_N = \cos \frac{2 \pi}{N} + i \sin \frac{2 \pi}{N} = e^{\frac{2 \pi}{N}i}
$$
この$\zeta_N$には重要な3つの性質があります。

- $\zeta_N^i = \zeta_N^{N + i}$
- $\sum^{N - 1}_{i = 0} \zeta_N^{i(j - k)} = \left\{ \begin{array}{ll} N & \text{if}\ j \equiv k \mod N \\ 0 & \text{otherwize} \end{array} \right.$
- 上記は$\zeta_N$を$\zeta_N^{-1}$と置き換えても成り立つ

実はこの$\zeta_N$は複素数でなくてもよく、この3つの性質を満たしていれば何でも良いのですが、それはまた後ほど。

DFTの話に戻りますが、もう一つ離散フーリエ逆変換(IDFT)というものもあります。定義は以下の通りです。
$$
f(x) = \frac{1}{N} \sum^{N - 1}_{i = 0} \hat{f}(\zeta^{-i}_N) x^i
$$
このIDFTを使うとDFTした式を復元することができます。また見たとおりDFTの定義と大して変わらないため、DFTが実装できればIDFTもすぐに実装できます。

# DFTとIDFTを使った多項式乗算

さて、ここで$f(x)$と$g(x)$の積にDFTを適用してみましょう。
$$
\begin{align*}
\widehat{f \cdot g} (x) &= \sum^{N - 1}_{i = 0} (f \cdot g) (\zeta_N^{i}) t^i \\
&= \sum^{N - 1}_{i = 0} f (\zeta_N^{i}) g (\zeta_N^{i}) t^i \\
\end{align*}
$$
このことから$\widehat{f \cdot g}(x)$は$f(x)$と$g(x)$それぞれにDFTを適用し、各項を乗算すれば求まることがわかります。IDFTを使えば元の$(f \cdot g)(x)$を求めることもできます。

つまり多項式乗算は以下の通りに計算することで求めることができます。

1. $f(x), g(x)$にDFTを適用し、$\hat f(x), \hat g(x)$を求める
2. $\hat f(x), \hat g(x)$を各項で乗算し、$\widehat{f \cdot g}(x)$を求める
3. $\widehat{f \cdot g}(x)$にIDFTを適用し、$(f \cdot g)(x)$を求める

さて計算量について考えましょう。1, 3の計算量はFFTを用いると$O(n \log n)$です。2はただのforで実装できるので計算量は$O(n)$です。よってDFT、IDFTを用いた多項式乗算の計算量は$O(n \log n)$になります。

# RustでのDFT, IDFTの実装

RustでのDFT, IDFTの単純実装を示します。複素数計算のために`num`クレートを使用していますがご了承ください。

`dft`はDFTとIDFTを兼ねています。`inverse`が`false`のときはDFT、`true`のときはIDFTを行います。FFTではないので計算量は$O(n^2)$のままです(むしろ定数倍が増えています)。

```rust
use num::Complex;
use std::f64::consts::PI;

fn main() {
    let f = vec![1, 2];
    let g = vec![3, 1, 4];

    let ans = mul_by_dft(&f, &g);
    println!("{:?}", ans);
}

fn mul_by_dft(f: &Vec<u32>, g: &Vec<u32>) -> Vec<u32> {
    let mut comp_f: Vec<Complex<f64>> = f.iter().map(|&x| Complex::new(x as f64, 0.0)).collect();
    let mut comp_g: Vec<Complex<f64>> = g.iter().map(|&x| Complex::new(x as f64, 0.0)).collect();

    let size = f.len() + g.len() - 1;
    comp_f.resize(size, Complex::new(0.0, 0.0));
    comp_g.resize(size, Complex::new(0.0, 0.0));

    let dft_f = dft(&comp_f, false);
    let dft_g = dft(&comp_g, false);

    let mut mul = vec![Complex::new(0.0, 0.0); size];
    for i in 0..size {
        mul[i] = dft_f[i] * dft_g[i];
    }

    let idft_mul = dft(&mul, true);

    idft_mul.iter().map(|x| x.re as u32).collect()
}

fn dft(f: &Vec<Complex<f64>>, inverse: bool) -> Vec<Complex<f64>> {
    let size = f.len();

    let mut res = vec![Complex::new(0.0, 0.0); size];

    for i in 0..size {
        let zeta = if !inverse {
            Complex::from_polar(1.0, 2.0 * PI * i as f64 / size as f64)
        } else {
            Complex::from_polar(1.0, -2.0 * PI * i as f64 / size as f64)
        };
        let mut now = Complex::new(1.0, 0.0);
        for j in 0..size {
            res[i] += f[j] * now;
            now *= zeta;
        }
    }

    if inverse {
        for i in 0..size {
            res[i] /= Complex::new(size as f64, 0.0);
        }
    }

    res
}
```

# まとめ

- 多項式乗算は普通に計算すると$O(n^2)$かかる
- 離散フーリエ変換を使うと多項式計算ができる
- 離散フーリエ変換は高速フーリエ変換を使うと$O(n \log n)$で計算できる

次回は高速フーリエ変換の概要と実装について解説します。
