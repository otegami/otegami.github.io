---
layout: post
title: "Static link と Dynamic Link"
date: 2023-06-06 21:17:25 +0800
categories: Red-Data-Tools
---

## 最初に
[Red Data Tools開発者に聞け！第6回　Apache ArrowのRubyバインディングに新機能を追加するシリーズ](https://www.youtube.com/watch?v=MEAOzM5UtEc) の中で、kou さんが解説してくださった内容を簡単にまとめるシリーズになります。
今回は Part1 で紹介された内容をまとめていきます。
- kou さんがサクッと解説してくださっている動画は、[こちら](https://youtu.be/MEAOzM5UtEc?t=623)になります。

## 遭遇した問題
今回は、Apache Arrow をビルドする中で遭遇した下記のエラーになります。
```console
% cmake -S cpp -B cpp.build --preset ninja-debug-maximal -DCMAKE_INSTALL_PREFIX=/tmp/local 
CMake Error at src/arrow/flight/CMakeLists.txt:48 (message):
  Must build Arrow statically to link Flight tests statically
```

### 何が起きているのか？
C++ のソースコードをビルドしようとして、必要なライブラリが正しくリンクできないために発生しています。
ただ、リンクが何かわかっておらず、よく分かっていないので次はリンクについてみていきます！

### リンクとは？
リンクは、C や C++ がライブラリを利用するためのプロセスです。Rubyでライブラリを利用するときには `require` を使いますが、C や C++ ではリンクというプロセスを通じてライブラリを利用します。
リンクには主に2つの形式があります。`Static link` と `Dynamic link` です。

### `Static link` と `Dynamic link`
Static link と Dynamic link の違いは、同じライブラリを複数のプログラムが利用する違いです。

####  Dynamic link
Ruby で考えると、複数の gem(ライブラリ)が同じ gem を利用する場合、例えば Rails で言うと 1 つの Active Support の実体を他の gem が共有して利用する状態が Dynamic link に相当します。なので Active Support のバージョンを更新すると、共有して利用していた他のライブラリ全てが影響を受けます。

#### Static link
一方、各 gem がそれぞれ Active Support を持っている状態は、Static link に近いと考えられます。それぞれのライブラリが独自のActive Supportを保持しているため、他のライブラリが保持している Active Support を更新しても、それが他のライブラリに影響を及ぼすことはありません。
- 実際に Ruby で Static Link をやっているライブラリっているの？ Rubygems がやっています。

### 対応策
正しくライブラリがリンクされるように修正する必要があります。
ただ、今回の Ruby バインディングの開発では、スタティックにリンクする必要や対象のライブラリは必要ではないので、エラーが起きているライブラリを使わずにビルドすることで回避します。具体的には下記の 2 つのオプションをつけます。
```
-DARROW_FLIGHT=OFF
-DARROW_BUILD_TESTS=OFF
```
```console
% cmake -S cpp -B cpp.build --preset ninja-debug-maximal -DARROW_FLIGHT=OFF -DARROW_BUILD_TESTS=OFF -DCMAKE_INSTALL_PREFIX=/tmp/local
CMake Error at src/arrow/flight/CMakeLists.txt:48 (message):
  Must build Arrow statically to link Flight tests statically
```
