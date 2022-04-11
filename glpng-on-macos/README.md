---
emoji: 🖼️
tags:
  - C言語
  - OpenGL
---

# macOS 上で glpng をコンパイルして動かす

## はじめに

今日から学校でプログラミング演習の授業が始まった。
プログラミング演習では OpenGL を用いてゲームを作るらしく、はじめは環境構築だった。
OpenGL の他に png を扱うための glpng というライブラリを導入する必要があったが、これを本家ソースから macOS 上でコンパイルしようとするとハマったので対処法をメモする。

## TL; DR

- Clang だとコンパイルできない
- ソースコードの修正が必要
- OpenGL と GLUT のフレームワークと一緒にコンパイルする必要がある

## gcc のインストール

macOS にはデフォルトで gcc が入っていない。正確にはコマンド自体は存在するのだが中身は Clang となっている。「自分のことを gcc だと思ってる精神異常 Clang」では glpng をコンパイルすることができないため、Homebrew で gcc をインストールする。なお、この gcc はコマンド名が`gcc`ではなく`gcc-<メジャーバージョン>`となるので注意。今回は`gcc-11`が入った。

```shell
brew install gcc

# Clangらしき記述がなければOK
gcc-11 --version
```

## ライブラリのソースを取得

どうやら本家サイトが落ちているらしく、[森北出版株式会社刊 「Springs of C」サポート web ページ](http://teacher.nagano-nct.ac.jp/ito/Springs_of_C/)からダウンロードした。`glpng.zip`をダウンロードすれば良い。
ダウンロードしたら適当な場所に展開しておく。

## `Makefile`とソースコードの修正

そのままではライブラリをコンパイルできないので`src/`下の`Makefile.LINUX`と`glpng.c`を修正する。

```diff
 # (省略)...

-CC	=	gcc
-CXX	=	g++
+CC	=	gcc-11
+CXX	=	g++-11
 CFLAGS	=	-pipe -Wall -W -O2 -fno-strength-reduce
 CXXFLAGS=	-pipe -Wall -W -O2 -fno-strength-reduce
-INCPATH	=	-I../include
+INCPATH	=	-framework OpenGL -framework GLUT -I../include
 AR	=	ar cqs
 RANLIB	=
 MOC	=	moc
```

```diff
 // (省略)...

+#include <GLUT/glut.h>
 #include <GL/glpng.h>
-#include <GL/gl.h>
 #include <stdlib.h>
 #include <math.h>
 #include "png/png.h"
```

修正した後、`src/`で`make -f Makefile.LINUX`すれば`lib/`に glpng のライブラリが生成されている。

## サンプルプログラムの修正

`Example/`に`Test.c`というサンプルプログラムがあるが、これもそのままでは動かないので修正する。

```diff
 // (省略)...

 #include <gl/glpng.h>
-#include <gl/glut.h>
+#include <GLUT/glut.h>
 #include <stdlib.h>

 // (省略)...

-void main() {
+int main(int argc, char **argv) {
 	pngInfo info;
 	GLuint  texture;

+	glutInit(&argc, argv);
 	glutInitDisplayMode(GLUT_DOUBLE | GLUT_RGB);
 	glutInitWindowSize(300, 300);
 	glutCreateWindow("glpng test");
```

## サンプルプログラムのコンパイル

サンプルプログラムも`gcc`でコンパイルしなければならず、更に OpenGL、GLUT といったフレームワークやコンパイルした glpng ライブラリなどのリンクが必要である。`Example/`で以下のコマンドでコンパイルできる。

```shell
gcc-11 -Wall Test.c -framework GLUT -framework OpenGL -I../include -L../lib -lglpng
```

生成された`a.out`を実行し画像が回転している表示がされれば成功。

## 最後に

以下の記事・本を参考にした。

- [森北出版株式会社刊 「Springs of C」サポート web ページ](http://teacher.nagano-nct.ac.jp/ito/Springs_of_C/)
- [Springs of C - 楽しく身につくプログラミング](https://www.amazon.co.jp/dp/4627849516)
