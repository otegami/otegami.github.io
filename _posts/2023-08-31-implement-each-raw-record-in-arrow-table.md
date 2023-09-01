---
layout: post
title: "Apache Arrow の Ruby バインディングに新機能を追加する 実装編 Part4"
date: 2023-08-31 20:28:45 +0800
categories: Red-Data-Tools
---

## 最初に
[Red Data Tools開発者に聞け！Apache Arrow の Ruby バインディングに新機能を追加するシリーズ](https://www.youtube.com/watch?v=n0NC_c88kDM) の中で、kou さんが解説してくださった内容を簡単にまとめるシリーズになります。
今回は Part14 で紹介された内容をまとめていきます。
- 動画で kou さんがサクッと解説してくださっています。動画で見たい方は上のリンクを参照してください！

## 記事の位置付け
1. 追加したい機能を Ruby で実装してイメージを掴む
2. Ruby が提供する C の API を利用してどのように実装していくのか学ぶ
3. 追加対象に関連するコードを読み解いていく
4. 追加機能の実装を行う <- **今回の記事はここ**

## 前回までのあらすじ
前回は、既存の `Arrow::Table#raw_records` の内部実装を読むことを通してコードベースの理解を深めるとともに実装イメージを掴みました。
今回は、実際に `Arrow::Table#raw_each_record` の実装を行っていきましょう！一気に作業をすると大変なのでおおまかな作業の流れをまとめて少しずつ実装していきます。

## おおまかな作業の流れ
下記の 4 つのステップで実装していこうと思います。
1. Ruby C API で Ruby のメソッドを定義する
2. Ruby C API で呼び出す C++ のメソッドを定義する
3. 列指向のデータ構造を行指向のデータ構造に変換するための RawRecordsProducer クラスを定義する
4. 変換結果を逐次的に yield して返す RawRecordsProducer#produce の実装する

### 1. Ruby C API で Ruby のメソッドを定義する
`Arrow::Table#raw_records` が定義されていたように `Arrow::Table#each_raw_record` を定義してあげたいので同様な形で書いてあげましょう！

```c++
// ruby/red-arrow/ext/arrow/arrow.cpp
auto cArrowTable = rb_const_get_at(mArrow, rb_intern("Table"));
rb_define_method(cArrowTable, "each_raw_record",
                 reinterpret_cast<>("TODO: C++ のメソッドを定義する必要あり")),
                 0);
```

Ruby C API の `rb_define_method` を利用して、`Arrow::Table` に `each_raw_record` を定義します。
これで Ruby からは、Arrow::Table#each_raw_record が見えるようになります。ただ、Ruby の世界で呼ばれた時に実際に実行したい C++ レイヤーのメソッドが登録されていないので何もしないメソッドになってしまっているので、次に C++ レイヤーのメソッドを定義してあげます。

### 2. Ruby C API で呼び出す C++ のメソッドを定義する
Ruby C API から実際に呼び出す C++ のメソッド `red_arrow::record_batch_each_raw_record` を定義してあげます。
まずはメソッドを定義したいだけなので Qnil を返すだけにしてあげます。

```c++
// ruby/red-arrow/ext/arrow/arrow.cpp
rb_define_method(cArrowTable, "each_raw_record",
                 reinterpret_cast<rb::RawMethod>(red_arrow::table_each_raw_record),
                 0);
```
```c++
// ruby/red-arrow/ext/arrow/raw-records.cpp
namespace red_arrow {
  namespace {
    VALUE
    table_each_raw_record(VALUE rb_table) {
      return Qnil;
    }
  }
}
```

実際にちゃんと呼ばれているのか少し試してみましょう！
変更をコンパイルして反映してあげてから、コンパイルされたオブジェクトファイルを参照してあげるようにして irb 上で確認します。

```cosnole
% make -C ext/arrow/
% bundle exec env DYLD_LIBRARY_PATH=$DYLD_LIBRARY_PATH:/tmp/local/lib irb -I ext/arrow
irb(main):001:0> require 'arrow'
irb(main):002:0> Arrow::Table.new(number: [1, 2, 3]).each_raw_record
=> nil
```

無事に `Arrow::Table#each_raw_record` が実行されていそうですね。nil も返ってきていそうです。
では、いよいよ行毎に yield して返すイテレーター部分の実装に入っていきましょう！

### 3. 列指向のデータ構造を行指向のデータ構造に変換するための RawRecordsProducer クラスを定義する
`red_arrow::table_each_raw_record` メソッドから `red_arrow::table_raw_records` メソッドのようにカラム指向のデータ構造を列指向のデータ構造に変換する責務をになった `RawRecordsProducer` Class クラスを呼び出すようにしてあげましょう。
- なぜ、新しく `RawRecordsProducer` を定義するかというと前回見たように `RawRecordsBuilder` では全てのデータを変換してから値を返す実装になっており、今回実装したいイテレーターとは根本的な役割が異なるため新しく定義をしています。

```c++
// ruby/red-arrow/ext/arrow/raw-records.cpp
namespace red_arrow {
  namespace {
    class RawRecordsProducer : private Converter, public arrow::ArrayVisitor {
    public:
      explicit RawRecordsProducer()
        : Converter(),
          record_(Qnil), // 返却する行データを入れるの利用する
          column_index_(0),
          row_offset_(0) {
      }
    }

    VALUE
    table_each_raw_record(VALUE rb_table) {
      auto garrow_table = GARROW_TABLE(RVAL2GOBJ(rb_table));
      auto table = garrow_table_get_raw(garrow_table).get();

      try {
        RawRecordsProducer producer; // メインとなるのはココ
      } catch (rb::State& state) {
        state.jump();
      }

      return Qnil;
    }
  }
}
```

ここまでで、核となる「変換結果を逐次的に yield して返す」ロジック部分を実装するためのお膳立てができましたね。
では最後のロジックの実装に移っていきます。

### 4. 変換結果を逐次的に yield して返す RawRecordsProducer#produce の実装する
今回の主体となる `RawRecordsProducer` クラスに対して変換し行毎に返すお仕事をしてもらうようのメソッド `RawRecordsProducer#produce` を定義していきます。

```c++
// ruby/red-arrow/ext/arrow/raw-records.cpp
namespace red_arrow {
  namespace {
    class RawRecordsProducer : private Converter, public arrow::ArrayVisitor {
    public:
      void produce(const arrow::Table& table) {
        rb::protect([&] {
          const auto n_columns = table.num_columns();
          const auto n_rows = table.num_rows();
          std::vector<int> chunk_indexes(n_columns);
          std::vector<int64_t> row_offsets(n_columns);
          for (int64_t i_row = 0; i_row < n_rows; ++i_row) {
            record_ = rb_ary_new_capa(n_columns);
            for (int i_column = 0; i_column < n_columns; ++i_column) {
              column_index_ = i_column;
              const auto chunked_array = table.column(i_column).get();
              auto& chunk_index = chunk_indexes[i_column];
              auto& row_offset = row_offsets[i_column];
              auto array = chunked_array->chunk(chunk_index).get();
              while (array->length() == row_offset) {
                ++chunk_index;
                row_offset = 0;
                array = chunked_array->chunk(chunk_index).get();
              }
              row_offset_ = row_offset;
              check_status(array->Accept(this),
                           "[table][each-raw-record]");
              ++row_offset;
            }
            rb_yield(record_);
          }

          return Qnil;
        });
      }
    }
  
    VALUE
    table_each_raw_record(VALUE rb_table) {
      auto garrow_table = GARROW_TABLE(RVAL2GOBJ(rb_table));
      auto table = garrow_table_get_raw(garrow_table).get();

      try {
        RawRecordsProducer producer;
        producer.produce(*table); 
      } catch (rb::State& state) {
        state.jump();
      }

      return Qnil;
    }
  }
}
```

一見 `produce` メソッドでやっていくことが複雑に見えますが、実は難しいことはしておらず大きく２つの仕事をしているだけです。(少し難しい部分としては、列のデータ量が多い場合は、 ChunkedArray という複数の要素から形成される場合も考慮する必要がある部分なのですが、今回はシンプルな解説にしたいので省いています。)
- 列指向のデータ構造から 1 行単位になるようにデータを抜き出して、`record_` に格納してあげる
- `record_` に格納したデータを Ruby C API が提供する `rb_yield` に渡してあげて、1 行単位のデータを返してあげる

ここまでで、`Arrow::Table#each_raw_record` の実装は完了です。実際に動くか確認してみましょう。

変更をコンパイルして反映してあげてから、コンパイルされたオブジェクトファイルを参照してあげるようにして irb 上で確認します。
ちゃんと行毎のデータがブロックに渡されて実行されていることがわかりました。問題なさそうですね。お疲れ様です！

```cosnole
% make -C ext/arrow/
% bundle exec env DYLD_LIBRARY_PATH=$DYLD_LIBRARY_PATH:/tmp/local/lib irb -I ext/arrow
irb(main):001:0> require 'arrow'
irb(main):002:0> Arrow::Table.new(number: [1, 2, 3]).each_raw_record { p _1 }
[1]
[2]
[3]
=> nil
```

### 最後に
「Apache Arrow の Ruby バインディングに新機能を追加する」を通して、「動的ライブラリとは」から「Rubyの仕組み」まで幅広い学びがありました。恥ずかしい話なのですが、普段は作成されたバインディングをライブラリ(gem)を通して利用するだけであったため何も知りませんでした。どこかの Rubyist が便利な機能を Ruby で使えるように道を用意してくれているから何も気にせずに私は利用できていたんだなと実感しました。

ただ、今回の体験を通して Ruby バインディングの作り方の基礎は理解できたのかなぁと思うので、これからは Rubyist が便利になる世界を作るのに貢献する側に回っていけたらと思っています！

最後に、全 15 回に及んでわかりやすく解説してくださった [kou](https://github.com/kou) さんと一緒に開発に取り組んでくださった [mterada1228](https://github.com/mterada1228) さんに改めてこの場でも感謝を伝えられたらと思います。本当にありがとうございました！！
