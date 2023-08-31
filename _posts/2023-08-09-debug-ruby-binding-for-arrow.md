---
layout: post
title: "Symbol not found in flat namespace - arrow.bundle(LoadError)"
date: 2023-08-09 07:37:19 +0800
categories: Red-Data-Tools
---

## 最初に
[Red Data Tools開発者に聞け！Apache Arrow の Ruby バインディングに新機能を追加するシリーズ](https://youtu.be/CNChc_WnAE0?list=PLKb0MEIU7gvQOIACKgdgKuAE7cMPDuTE6&t=1550) の中で、kou さんが解説してくださった内容を簡単にまとめるシリーズになります。
今回は Part12 で紹介された内容をまとめていきます。
- 動画で kou さんがサクッと解説してくださっています。動画で見たい方は上のリンクを参照してください！

## 遭遇した問題
今回は、Red Arrow のテスト実行時に遭遇した下記のエラーになります。
具体的には、対象のシンボルが見つからず LoadError になっている状態です。
```console
% pwd
/dev/project/arrow/ruby/red-arrowdev/project/arrow/ruby/red-arrow
red-arrow % bundle exec env DYLD_LIBRARY_PATH=$DYLD_LIBRARY_PATH:/tmp/local/lib $(rbenv which ruby) test/run-test.rb
/dev/project/arrow/ruby/red-arrow/lib/arrow/loader.rb:146:in `require': dlopen(/dev/project/arrow/ruby/red-arrow/ext/arrow/arrow.bundle, 0x0009): symbol not found in flat namespace '__ZN9red_arrow21table_each_raw_recordEm' - /dev/project/arrow/ruby/red-arrow/ext/arrow/arrow.bundle (LoadError)
        from /dev/project/arrow/ruby/red-arrow/lib/arrow/loader.rb:146:in `require_extension_library'
        from /dev/project/arrow/ruby/red-arrow/lib/arrow/loader.rb:31:in `post_load'
        from /.rbenv/versions/3.2.0/lib/ruby/gems/3.2.0/gems/gobject-introspection-4.1.6/lib/gobject-introspection/loader.rb:49:in `block in load'
        from /.rbenv/versions/3.2.0/lib/ruby/gems/3.2.0/gems/gobject-introspection-4.1.6/lib/gobject-introspection/loader.rb:636:in `prepare_class'
        from /.rbenv/versions/3.2.0/lib/ruby/gems/3.2.0/gems/gobject-introspection-4.1.6/lib/gobject-introspection/loader.rb:41:in `load'
        from /.rbenv/versions/3.2.0/lib/ruby/gems/3.2.0/gems/gobject-introspection-4.1.6/lib/gobject-introspection/loader.rb:25:in `load'
        from /dev/project/arrow/ruby/red-arrow/lib/arrow/loader.rb:24:in `load'
        from /dev/project/arrow/ruby/red-arrow/lib/arrow.rb:29:in `<module:Arrow>'
        from /dev/project/arrow/ruby/red-arrow/lib/arrow.rb:25:in `<top (required)>'
        from /dev/project/arrow/ruby/red-arrow/test/helper.rb:18:in `require'
        from /dev/project/arrow/ruby/red-arrow/test/helper.rb:18:in `<top (required)>'
        from test/run-test.rb:67:in `require_relative'
        from test/run-test.rb:67:in `<main>'
```

## 対応策
エラーを見ると `__ZN9red_arrow21table_each_raw_recordEm` のようにマングリングされたシンボル情報が出ているので下記のコマンド(c++filt)でデマングルしてあげると、具体的に見つからないシンボル名を見ることができます。今回は、`red_arrow::table_each_raw_record` が見つかっていないようです。
```console
% c++filt
__ZN9red_arrow21table_each_raw_recordEm
red_arrow::table_each_raw_record(unsigned long)
```

ソースコード内で `red_arrow::table_each_raw_record` に関する部分を探すと `red_arrow::table_each_raw_records` のようにタイポしているのを発見しました。タイポを修正し再度コンパイルを行った後に実行してあげると無事に実行できました。

```console
% bundle exec env DYLD_LIBRARY_PATH=$DYLD_LIBRARY_PATH:/tmp/local/lib $(rbenv which ruby) -I ext/arrow sample.rb
[1, false]
[2, true]
[3, false]
[4, true]
```

ここからは、マングリングって何？と気になる人 or 上記で解決しない場合のもう少ししっかりしたデバッグ知識を得たい人向けになります。

### マングリングとは？
C++はRubyと異なり、ネームスペース、クラス、関数のオーバーロードなどで同じ名前を持つ要素が存在します。コンパイルしてバイナリに変換する際、これらの名前はリンカーによって区別される必要があります。

ここで出てくるのが、「マングリング（Name Mangling）」です。マングリングは、コンパイラがバイナリレベルで名前を一意にするための自動変換を指します。この変換により、異なるネームスペース、クラス、オーバーロードされた関数も一意の名前を持つようになるのです。（バイナリ内の関数名などは「シンボル」として呼ばれます。）

マングリングを通じて、リンカーは関数や変数を正確に識別し、適切にリンクすることができるのです。

ref: [https://www.ibm.com/docs/ja/i/7.5?topic=linkage-name-mangling-c-only](https://www.ibm.com/docs/ja/i/7.5?topic=linkage-name-mangling-c-only)

### デマングルとは？
マングリングされたシンボルを元の名前に戻す変換になります。`c++filt` を利用することでデマングルを行うことができます。

ref: [https://www.ibm.com/docs/ja/xl-c-and-cpp-aix/16.1?topic=utilities-demangling-compiled-c-names](https://www.ibm.com/docs/ja/xl-c-and-cpp-aix/16.1?topic=utilities-demangling-compiled-c-names)

## 対応策（詳細編）
デマングルすることで `red_arrow::table_each_raw_record` に問題があることがわかりました。
では本当にこのシンボルがバイナリファイル(`arrow.bundle`)に存在しないのかを確認してみましょう！

### バイナリファイル内からシンボルを探す
バイナリファイルは、人間には読みづらいので `nm` コマンドでファイル内のシンボル情報を表示してもらいましょう。
- 今回は、デマングルも同時に行いたいので `-C` オプションをつけてあげましょう。

下記のような結果が出てきましたので、それぞれが意味するところをみていきましょう！

```console
% nm -C ext/arrow/arrow.bundle | grep red_arrow::table_each_raw_record
                 U red_arrow::table_each_raw_record(unsigned long)
00000000000077d8 T red_arrow::table_each_raw_records(unsigned long)
```

左からアドレス(シンボルのオフセット)シンボルのタイプ、シンボル名を表しています。
では、`U` と `T` はそれぞれどういったシンボルのタイプなのかを確認してみましょう。
`man` コマンドで `nm` の説明を見ていきます。

```
% man nm
```

どうやら、U (undefined) は未定義状態、T (text section symbol)は、テキストシンボルであることがわかりました。
つまり、`red_arrow::table_each_raw_record` は未定義状態であり、`red_arrow::table_each_raw_records` は定義されていることがわかります。
なので、今回はタイポしていそうなことがわかります。

### 個人的に思ったこと
バイナリファイル(オブジェクトファイル)をデバックするなどと聞くと身構えてしまいますが、人間が理解しやすい形にツールを利用し変換してあげれば、簡単にデバックできることを学びました。デバックができればあとは Try and Error
