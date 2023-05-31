---
layout: post
title: "Apache Arrow の Ruby バインディングに新機能を追加する 準備編"
date: 2023-05-31 22:59:18 +0800
categories: Apache Arrow
---

## 何をやりたいか
- Apache Arrow の Rubyバインディング（Red Arrow）に新機能を追加したい。
- 具体的にはこちらの [[Ruby] Add Arrow::RecordBatch#each_raw_record](https://github.com/apache/arrow/issues/33749)を解決したい。

### 登場人物
- Apache Arrow
	- 実装言語: C++ 
	- 効率的なデータ分析を可能にする、言語に依存しない列指向メモリフォーマットを提供するツール
- Red Arrow
	- 実装言語: Ruby, C++
	- Apache Arrow を Ruby  から使えるようにする Ruby の拡張ライブラリ
- CMake
	- CMakeはクロスプラットフォームで柔軟性の高いビルドシステムです
	- Apache Arrow C++ のコンフィグレーションに利用されます
- Meson
	- 高速でユーザーフレンドリーなクロスプラットフォームのビルドシステム
	- Apache Arrow GLib のコンフィグレーションに利用されます
- Ninja
	- 高速なビルドシステムで、大規模なソフトウェアプロジェクトのコンパイルを効率的に行います
	- CMakeやMesonなどのビルドシステムと組み合わせて、ビルドの設定と実行を行います
	- Apache Arrow C++ と Apache Arrow GLib のビルドに利用されます

## 準備としてやること
Red Arrow の機能追加に着手するための開発環境の構築を行なっていく。
下記のライブラリに依存しているのでインストールを行なっていく。
1. Apache Arrow C++ のインストール
2. Apache Arrow GLib のインストール
3. Red Arrow のビルド

Ref: https://github.com/apache/arrow/tree/main/ruby/red-arrow#development

### Apache Arrow C++ のインストール
基本的には下記に記されている手順通りに Arrow C++ のビルドを進めていく。
+α で教えていただいこともメモとして追記します。
- https://arrow.apache.org/docs/developers/cpp/building.html

インストールする際には大きく 3 つのステップがあります。
1. コンフィグレーション
	- ビルドに必要な依存パッケージの準備やビルド手順を定義したファイルを生成する
2. ビルド
	- コンフィグレーションで生成されたファイルを元にビルドする
	- 具体的には、ソースコードのコンパイルとリンクをする
3. インストール
	- ビルドした成果物をインストールする

#### 事前準備
ビルドに必要なライブラリを Homebrew を利用していインストールします。
```console
% git clone https://github.com/apache/arrow.git
% cd arrow
% brew update && brew bundle --file=cpp/Brewfile
```

#### コンフィグレーション
ビルドに必要な依存パッケージの準備（場所の把握やインストール）やビルド手順を定義したファイルを生成していきます。
今回は、普通にコンフィグレーションするだけではなくいくつかのオプションを追記しています。
```console
% cmake -S cpp -B ./cpp.build --preset ninja-debug-maximal -DARROW_CUDA=OFF -DARROW_SKYHOOK=OFF -DCMAKE_INSTALL_PREFIX=/tmp/local
```

- `-S cpp `
	- コンフィグレーションする際に利用するソースディレクトリを指定
	- 今回は、`cpp` 配下を利用し、コンフィグレーションしていきます
- `-B ./cpp.build`
	- コンフィグレーションしビルドする際に利用する成果物を配置するディレクトリを指定
	- `xxx.build` の命名は、[kou](https://github.com/kou) さんのオススメです
		- ビルド元がわかりやすい && ビルド用ディレクトリであることがわかりやすい
- `--preset ninja-debug-maximal`
	- どのようにコンフィグレーションするかを指定できるオプションが提供されています
	- 今回は、ビルドツールに Ninja を利用し、全てを有効化にしたデバッグビルドを行う設定にしています
- `-DCMAKE_INSTALL_PREFIX=/tmp/local`
	- ビルドした際の成果物のインストール先を指定します
	- 今回は、`/tmp/local`　を指定しています
- `-DARROW_CUDA=OFF -DARROW_SKYHOOK=OFF`
	- Arrow C++ が提供する特定の機能に対するフラグになります
	- 今回は、`CUDA`　との連携機能と `Skyhook` 機能を無効にしています
		- ビルド時にエラーになったため（今回拡張したい機能では不要なため許容しています）
	- ref: https://arrow.apache.org/docs/developers/cpp/building.html#optional-components

#### ビルド
コンフィグレーションで準備されたビルド環境を元にビルドを行なっていきます。
```console
% cmake --build ./cpp.build
```
- `--build ./cpp.build`
	- ビルド対象のディレクトリを指定します
	- 今回は、`cpp.build` 配下を指定しています
		- なぜなら、コンフィグレーション時に `-B`  オプションで指定したため

#### インストール
ビルドされた成果物をインストールしていきます。
今回は、コンフィグレーション時に指定した `-DCMAKE_INSTALL_PREFIX=/tmp/local` にインストールします。
```
% ninja -C cpp.build install
```
- `-C cpp.build`
	- `Ninja` を実行するディレクトリを指定できます
	- 今回は、ビルドした `cpp.build` を対象にしてインストールをします

ここまでで、Apache Arrow C++ のインストールが完了です。お疲れ様です！

### Apache Arrow GLib のインストール
基本的には、Arrow C++ と同様の下記の 3 つのステップでインストールしていきます。
下記の情報を元に進めていきます。
- https://github.com/apache/arrow/blob/main/c_glib/README.md

1. コンフィグレーション
2. ビルド
3. インストール

#### 事前準備
ビルドに必要なライブラリを Homebrew を利用していインストールします。
```
% brew update && brew bundle --file=c_glib/Brewfile
```

#### コンフィグレーション
ビルドに必要な依存パッケージの準備（場所の把握やインストール）やビルド手順を定義したファイルを生成していきます。
今回は、普通にコンフィグレーションするだけではなくいくつかのオプションを追記しています。
```console
% meson setup ./c_glib.build ./c_glib --prefix /tmp/local --debug --pkg-config-path /tmp/local/lib/pkgconfig
```
- `setup ./c_glib.build ./c_glib` 
	- setup は、ビルド用の成果物を配置するディレクトリと、コンフィグレーションする元になるディレクトリを指定します
- `--prefix /tmp/local`
	- ビルドした際の成果物のインストール先を指定します
- `--debug` 
	- ビルド時にデバック情報を生成するようにコンパイラに指示します
- `--pkg-config-path /tmp/local/lib/pkgconfig`
	- pkg-config がライブラリを探す際に探索するパスを指定できます
	- 今回は、依存ライブラリとして Arrow C++ を利用するので、前述のインストール先を参照しています

#### ビルド
`Meson`  でコンフィグレーションされたビルド環境を元にビルドを行なっていきます。
```console
% ninja -C c_glib.build
```
- `-C cpp.build`
	- `Ninja` を実行するディレクトリを指定できます
	- 今回は、コンフィグレーションで準備したビルドディレクトリ  `c_glib.build`  を対象にして、ビルドします

#### インストール
ビルドされた成果物をインストールしていきます。
今回は、コンフィグレーション時に指定した `/tmp/local` にインストールします。
```console
% ninja -C c_glib.build install
```
- オプションに関しては、上記で説明しているので省くね

ここまでで、Apache Arrow GLib のインストールが完了です。お疲れ様です！

#### Red Arrow のビルド
前述の Apache Arrow C++ と Apache Arrow GLib のインストール と同様の手順で進めていきます。
今回は、インストールする必要がないので、2 ステップになります。
1. コンフィグレーション
2. ビルド

#### 事前準備
Red Arrow のビルドに必要な gem をインストールしていきます。
```console
% cd ruby; bundle install;
% cd red-arrow; bundle install 
```

#### コンフィグレーション
`ext/arrow/` 配下の `extconf.rb` を元にコンフィグレーションを行います。
```console
% cd ext/arrow; ruby extconf.rb
```

#### ビルド
コンフィグレーションによって生成された Makefile を元にビルドを行います。
```console
% make
```

#### テスト
最後に Red Arrow が正常にビルドされたかをテストします。
```console
% pwd
/arrow/ruby/red-arrow
% bundle exec rake test
```

無事にテストが完了したら、Red Arrow のビルドですおめでとうございます！
