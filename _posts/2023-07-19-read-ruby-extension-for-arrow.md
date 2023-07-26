---
layout: post
title: "Apache Arrow の Ruby バインディングに新機能を追加する 実装編 Part2"
date: 2023-07-19 06:56 +0800
categories: Red-Data-Tools
---

## 最初に
[Red Data Tools開発者に聞け！第13回 Apache Arrow の Ruby バインディングに新機能を追加するシリーズ](https://youtu.be/eZGLDFr4s90?t=1225) の中で、kou さんが解説してくださった内容を簡単にまとめるシリーズになります。
今回は Part7 で紹介された内容をまとめていきます。
- 動画で kou さんがサクッと解説してくださっています。動画で見たい方は上のリンクを参照してください！

## 記事の位置付け
1. 追加したい機能を Ruby で実装してイメージを掴む
2. Ruby が提供する C の API を利用してどのように実装していくのか学ぶ <- **今回の記事はここ**
3. 追加対象に関連するコードを読み解いていく
4. 追加機能の実装を行う

## 前回までのあらすじ
前回は、「追加したい機能を Ruby で実装してイメージを掴む」ためにイテレータを Ruby レベルではどのように実装するのかをみていきました。今回は、前回実装したコードをもとに Ruby が提供する C の API を利用してどのように実装していくのかを関連するコードを確認しながらみていきましょう。

```ruby
require 'arrow'

class Arrow::Table
  def each_raw_record
    raw_records.each do |row|
      yield row
    end
  end
end
```

## C レイヤーでの実装の流れを掴む
Ruby レベルのイテレータ実装では、大きく 3 つのことを行っていました。

- 拡張ライブラリを `require` して読み込む
```
require 'arrow'
```

- メソッドを定義する
```ruby
  def each_raw_record
    #...
  end
```

- `yield` してイテレートする
```ruby
  raw_records.each do |row|
    yield row
  end
```

では次に Ruby が提供する C の API を利用してどのように上記の 3 つを実現するのかを見ていきましょう！

### 拡張ライブラリを `require` して読み込む
Ruby は拡張ライブラリを `require` してロードする際に「Init_ライブラリ名」という関数を自動で実行するようになっています。
- Ruby の公式ドキュメント「[Rubyの拡張ライブラリの作り方](https://docs.ruby-lang.org/en/3.2/extension_ja_rdoc.html#label-C-E3-82-B3-E3-83-BC-E3-83-89-E3-82-92-E6-9B-B8-E3-81-8F)」を参照

実際に、`red-arrow` ではどうなっているのか確認して見ると、`Init_arrow` 関数が定義されています。
つまり、Ruby レベルで `require arrow` をすると、`Init_arrow` が実行されるという流れになります。
```cpp
extern "C" void Init_arrow() {
  // ...
}
```
- ref: [https://github.com/apache/arrow/blob/main/ruby/red-arrow/ext/arrow/arrow.cpp#L68](https://github.com/apache/arrow/blob/main/ruby/red-arrow/ext/arrow/arrow.cpp#L68)

では次に、C レイヤーで「メソッドを定義する」のはどのように行うのかをみていきます。

### メソッドを定義する

Ruby の C API は、rb で始まる関数を提供しています。
例えば、実際に [Arrow の拡張機能のコード](https://github.com/apache/arrow/blob/main/ruby/red-arrow/ext/arrow/arrow.cpp#L68-L121)を見てみると下記のメソッドが利用されています
- `rb_const_get`
- `rb_define_method`
- `rb_const_get_at`
- `rb_intern`

今回注目したいのは、`rb_define_method` です。
これは、C レベルで Ruby の関数を定義するために Ruby が用意した C API になります。
実際に利用例を見てみると、`Arrow::Table` に `raw_records` メソッドを定義しているのがわかります。

```c++
auto cArrowTable = rb_const_get_at(mArrow, rb_intern("Table"));
rb_define_method(cArrowTable, "raw_records",
                 reinterpret_cast<rb::RawMethod>(red_arrow::table_raw_records),
                 0);
```

ref: [Rubyt C API: rb_define_method](https://github.com/ruby/ruby/blob/master/doc/extension.rdoc#method-and-singleton-method-definition-)

では次に、C レイヤーで「`yield` してイテレートする」のはどのように行うのかをみていきます。

## yield してイテレートする

実は、Ruby が `rb_define_method` C API を提供したいたのと同様に `yield` に関しても、`rb_yield` という API を提供してくれています。
なので、こちらを利用してあげれば `yield` して結果を逐次的に返すことができます。

ref: [Rubyt C API: rb_yield](https://github.com/ruby/ruby/blob/master/doc/extension.rdoc#control-structure-)

ここまでで大まかに C API を利用した実装の流れが追えたので、次の記事では実際に実装な知識を得るために内部実装を読み解いていきます。
