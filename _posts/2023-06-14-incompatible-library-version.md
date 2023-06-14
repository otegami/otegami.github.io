---
layout: post
title: "Incompatible Library Version Error"
date: 2023-06-14 20:54 +0800
categories: Red-Data-Tools
---

## 最初に
[Red Data Tools開発者に聞け！第12回 Apache Arrow の Ruby バインディングに新機能を追加するシリーズ](https://www.youtube.com/watch?v=63vTddvnvQw) の中で、kou さんが解説してくださった内容を簡単にまとめるシリーズになります。
今回は Part7 で紹介された内容をまとめていきます。
- kou さんがサクッと解説してくださっている動画は、[こちら](https://www.youtube.com/watch?v=63vTddvnvQw&t=320s)になります。

## 遭遇した問題
今回は、テスト実行時に発生した `incompatible library version` エラーになります。
```console
% bundle exec env DYLD_LIBRARY_PATH=$DYLD_LIBRARY_PATH:/tmp/local/lib $(rbenv which ruby) test/run-test.rb
/Users/otegami/dev/project/arrow/ruby/red-arrow/lib/arrow/loader.rb:146:in `require': incompatible library version - /Users/otegami/dev/project/arrow/ruby/red-arrow/ext/arrow/arrow.bundle (LoadError)
```

## 対応策
`arrow.bundle` をビルドした Ruby のバージョンでテストを実行することで解決できます。
```console
% bundle exec env DYLD_LIBRARY_PATH=$DYLD_LIBRARY_PATH:/tmp/local/lib $(rbenv which ruby) test/run-test.rb
```

### 何が起きているのか？
実行している `Ruby` が `arrow.bundle` が使っている`libruby`に対し、「ABI」（アプリケーションバイナリインターフェース）の互換性がなくエラーとなっている。
- `ABI` とは
  - バイナリレベルでの相互作用を規定しているもの（バイナリレベルのインターフェイス）
- `libruby` とは
  - Rubyプログラミング言語のコア機能を提供するライブラリです
  - 普段使っている ruby コマンドは、libruby を呼び出してスクリプトを実行しているだけである

## 互換性のある `libruby` は何なのか確認する
`otool`コマンドを利用して、`arrow.bundle` と互換性のある libruby を調査します。
- `otool` は、実行可能ファイルフォーマットである `Mach-O`（Mach Object）ファイル形式を調査するためのコマンドです。
- `Mach-O` は、コンパイラが生成するオブジェクトファイル並びに実行ファイルのフォーマットみたいです。（理解できていないので後日調べる）
- `-L` は引数に与えられたライブラリが利用する共有ライブラリの一覧を表示します

```console
% otool -L ext/arrow/arrow.bundle
ext/arrow/arrow.bundle:
  /Users/otegami/.rbenv/versions/3.2.0/lib/libruby.3.2.dylib (compatibility version 3.2.0, current version 3.2.0)
```

上記の結果から、`libruby.3.2.dylib` に依存していることがわかります。
では、現在利用している `Ruby` はどのパージョンの `libruby` に依存しているのか確認してみます。
```console
% otool -L $(rbenv which ruby)
/Users/otegami/.rbenv/versions/3.0.3/bin/ruby:
  /Users/otegami/.rbenv/versions/3.0.3/lib/libruby.3.0.dylib (compatibility version 3.0.0, current version 3.0.3)
```

`libruby.3.0.dylib` に依存していることがわかります。`libruby` のバージョンは `Ruby` のバージョンと互換性があるので、
今回は、`libruby.3.2.dylib` を利用する `Ruby v3.2.0` で実行してあげるとよさそうです。
```console
% rbenv global 3.2.0
% ruby -v
ruby 3.2.0 (2022-12-25 revision a528908271) [arm64-darwin20]
% bundle exec env DYLD_LIBRARY_PATH=$DYLD_LIBRARY_PATH:/tmp/local/lib ruby test/run-test.rb
Loaded suite test
Started
......................................
Finished in 1.123513 seconds.
-----------------------------------------------------------------------------------------------------------------------------------
1610 tests, 1610 assertions, 0 failures, 0 errors, 0 pendings, 0 omissions, 0 notifications
100% passed
-----------------------------------------------------------------------------------------------------------------------------------
1433.01 tests/s, 1433.01 assertions/s
```

無事にテストが通りました！
今までは、incompatible library version エラーが発生した場合、手探りで Ruby のバージョンを変更して問題を解決していましたが、今後は、libruby のバージョンを確認し、適切なバージョンの Ruby を選択することで問題を回避できます。
また、他のライブラリでも依存関係を調べながら解決できるでしょう！
