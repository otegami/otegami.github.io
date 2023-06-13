---
layout: post
title: "DYLD Environment Variables"
date: 2023-06-13 19:15 +0800
categories: Red-Data-Tools
---

## 最初に
[Red Data Tools開発者に聞け！第11回 Apache Arrow の Ruby バインディングに新機能を追加するシリーズ](https://www.youtube.com/watch?v=ClaRIi5fBag) の中で、kou さんが解説してくださった内容を簡単にまとめるシリーズになります。
今回は Part6 で紹介された内容をまとめていきます。
- kou さんがサクッと解説してくださっている動画は、[こちら](https://youtu.be/ClaRIi5fBag?t=748)になります。

## 遭遇した問題
今回は、Red Arrow のテスト実行時に遭遇した下記のエラーになります。
```console
% pwd
/arrow/ruby/red-arrow
% bundle exec rake test
(null)-WARNING **: Failed to load shared library 'libarrow-glib.1300.dylib' referenced by the typelib: dlopen(libarrow-glib.1300.dylib, 0x0009): tried: 'libarrow-glib.1300.dylib' (no such file), '/usr/local/lib/libarrow-glib.1300.dylib' (no such file), '/usr/lib/libarrow-glib.1300.dylib' (no such file), '/Users/otegami/dev/project/arrow/ruby/red-arrow/libarrow-glib.1300.dylib' (no such file), '/usr/local/lib/libarrow-glib.1300.dylib' (no such file), '/usr/lib/libarrow-glib.1300.dylib' (no such file)
```

## 対応策
下記のように実行することで問題を解決できます。その理由については以下で詳しく説明します。
```console
% bundle exec env DYLD_LIBRARY_PATH=$DYLD_LIBRARY_PATH:/tmp/local/lib $(rbenv which ruby) test/run-test.rb
```

### 何が起きているのか？
`DYLD_LIBRARY_PATH` を探索したが、共有ライブラリ（動的ライブラリ）の`libarrow-glib.1300.dylib` が見つからず、エラーになっています。
- `DYLD_LIBRARY_PATH` とは、MacOS が共有ライブラリのパスを管理する環境変数になります。

今回は`libarrow-glib.1300.dylib`が存在するパスを指定する必要がありそうです。

### DYLD_LIBRARY_PATH に `libarrow-glib.1300.dylib` のパスを指定する
```console
% ls /tmp/local/lib | grep libarrow-glib.1300.dylib
libarrow-glib.1300.dylib
```
`/tmp/local/lib` 配下に `libarrow-glib.1300.dylib` は存在することがわかったので、環境変数を指定してテストを実行してみます。
```console
% DYLD_LIBRARY_PATH=$DYLD_LIBRARY_PATH:/tmp/local/lib bundle exec rake test
(null)-WARNING **: Failed to load shared library 'libarrow-glib.1300.dylib' referenced by the typelib: dlopen(libarrow-glib.1300.dylib, 0x0009): tried: 'libarrow-glib.1300.dylib' (no such file), '/usr/local/lib/libarrow-glib.1300.dylib' (no such file), '/usr/lib/libarrow-glib.1300.dylib' (no such file), '/Users/otegami/dev/project/arrow/ruby/red-arrow/libarrow-glib.1300.dylib' (no such file), '/usr/local/lib/libarrow-glib.1300.dylib' (no such file), '/usr/lib/libarrow-glib.1300.dylib' (no such file)
```

`DYLD_LIBRARY_PATH` の探索パスに指定した `/tmp/local/lib/` が含まれていなさそうです。
どうやら環境変数での指定が効いていなさそうです。なぜなんでしょうか？

### MacOS 上で `DYLD_LIBRARY_PATH` は特別な環境変数である
試しに、Ruby が `DYLD_LIBRARY_PATH` 環境変数を認識しているか確認してみましょう。
```console
% DYLD_LIBRARY_PATH=/tmp/local/lib ruby -e 'pp ENV' | grep DYLD_LIBRARY_PATH
# 何も参照されず、存在しない
```

環境変数名を `XDYLD_LIBRARY_PATH` に変更してあげると認識します。どうやら環境変数 `DYLD_LIBRARY_PATH` が特別な動きをしているように見えます。
```console
% XDYLD_LIBRARY_PATH=/tmp/local/lib ruby -e 'pp ENV' | grep XDYLD_LIBRARY_PATH
 "XDYLD_LIBRARY_PATH"=>"/tmp/local/lib",
```

実は、DYLD_LIBRARY_PATH はサブプロセスには継承されないという仕様が存在します。
なぜなら、実行時にどのライブラリを読み込むかを指定できるため、仮に指定できてしまうと実行中のプログラムをインターセプトできてしまうような問題が起きてしまいます。
下記の Doc を参考にすると MacOS 側で読み込める用に設定を変えることもできそうですが、セキュリティ的によろしくなさそうなので、今回は利用しません。
- ref: [DYLD_LIBRARY_PATH and make](https://developer.apple.com/forums/thread/9233)
- ref: [Allow DYLD Environment Variables Entitlement](https://developer.apple.com/documentation/bundleresources/entitlements/com_apple_security_cs_allow-dyld-environment-variables)
- ref: [Disable Library Validation Entitlement](https://developer.apple.com/documentation/bundleresources/entitlements/com_apple_security_cs_disable-library-validation)

では、テストをどうやってメインプロセスで実行するかを考えるために、まず現状どのように実行されているのかをみていきましょう。

### ruby command はサブプロセスで実行されている？
ruby command の実体を確認してみましょう。次に示すファイルがそれに相当すると思われます。
- `type` command は、`which` commnad よりも幅広い情報を提供します。（エイリアスが使われている場合は、その内容を報告してくれるなど）
```console
% type ruby
ruby is /Users/otegami/.rbenv/shims/ruby
```

次にファイルの中身がなんなのかを `file` command で確認してみます。
どうやら Shell Script で書かれているようです。
```console
% file =ruby
/Users/otegami/.rbenv/shims/ruby: Bourne-Again shell script text executable, ASCII text
```

ファイルの中身は下記のようになっています。
どうやらこれは rbenv をラップして実行する Shell Script のようです。
これでは、サブプロセスになってしまうので、直接実行してあげるようにしてあげる必要がありそうです。
```shell
% cat =ruby
#!/usr/bin/env bash
set -e
[ -n "$RBENV_DEBUG" ] && set -x

program="${0##*/}"
if [ "$program" = "ruby" ]; then
  for arg; do
    case "$arg" in
    -e* | -- ) break ;;
    */* )
      if [ -f "$arg" ]; then
        export RBENV_DIR="${arg%/*}"
        break
      fi
      ;;
    esac
  done
fi

export RBENV_ROOT="/Users/otegami/.rbenv"
exec "/opt/homebrew/bin/rbenv" exec "$program" "$@"
```

直接実行するために `rbenv which ruby` の実体を確認してみると、バイナリのようです。
つまり、この実行可能なバイナリを直接実行すれば、シェルによって実行されるので環境変数をそのまま渡せそうです。
```console
% file =$(rbenv which ruby)
/Users/otegami/.rbenv/versions/3.2.0/bin/ruby: Mach-O 64-bit executable arm64
```

無事に `DYLD_LIBRARY_PATH` が参照でき、テストが通りました！
```console
% bundle exec env DYLD_LIBRARY_PATH=$DYLD_LIBRARY_PATH:/tmp/local/lib $(rbenv which ruby) test/run-test.rb
......................................
Finished in 1.032504 seconds.
-----------------------------------------------------------------------------------------------------------------------------------
1610 tests, 1610 assertions, 0 failures, 0 errors, 0 pendings, 0 omissions, 0 notifications
100% passed
-----------------------------------------------------------------------------------------------------------------------------------
1559.32 tests/s, 1559.32 assertions/s
```

念の為、環境変数 `DYLD_LIBRARY_PATH` に指定した値を確認してみると、無事に参照できていそうです！
```console
% bundle exec env DYLD_LIBRARY_PATH=$DYLD_LIBRARY_PATH:/tmp/local/lib $(rbenv which ruby) -e 'pp ENV' | grep DY
"DYLD_LIBRARY_PATH"=>":/tmp/local/lib",
```
