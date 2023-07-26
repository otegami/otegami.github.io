---
layout: post
title: "Apache Arrow の Ruby バインディングに新機能を追加する 実装編 Part3"
date: 2023-07-26 19:23:15 +0800
categories: Red-Data-Tools
---

## 最初に
[Red Data Tools開発者に聞け！Apache Arrow の Ruby バインディングに新機能を追加するシリーズ](https://www.youtube.com/playlist?list=PLKb0MEIU7gvQOIACKgdgKuAE7cMPDuTE6) の中で、kou さんが解説してくださった内容を簡単にまとめるシリーズになります。
今回は Part9 で紹介された内容をまとめていきます。
- 動画で kou さんがサクッと解説してくださっています。動画で見たい方は上のリンクを参照してください！

## 記事の位置付け
1. 追加したい機能を Ruby で実装してイメージを掴む
2. Ruby が提供する C の API を利用してどのように実装していくのか学ぶ
3. 追加対象に関連するコードを読み解いていく <- **今回の記事はここ**
4. 追加機能の実装を行う

## 前回までのあらすじ
前回は、Ruby の拡張ライブラリを実装する際の簡単なお作法や C API の利用方法を押さえていく中で、Ruby の C API を利用して大まかにどのような流れで実装していくのかを見ていきました。
前回でおおまかな流れは抑えられたので、今回は実装に必要な知識を得るために内部実装を読んでいきましょう！

## 内部実装を読んでいくための入り口を決めよう

今回は `Arrow::Table#raw_records` イテラブルな結果を返すバージョンの `Arrow::Table#raw_each_record` を実装したいので、`Arrow::Table#raw_records` を読んでいきたいと思います！

すでに「Part1」で確認済みですが、実際にどのような結果が返ってくるのかを念の為おさらいをしていきます。
`Arrow::Table` で表現されるような表形式のデータを `Arrow::Table#raw_records` を利用することで行形式のデータに変換した値が返ってきます。メソッドの動作についての理解が深まったと思うので、次はその実装を詳しく見ていきましょう！
```ruby
require 'arrow'

table = Arrow::Table.new(number: [1, 2, 3], string: ['a', 'b', 'c'])
p table
#<Arrow::Table:0x1050f2c00 ptr=0x106cfcbb0>
        number  string
0            1  a     
1            2  b     
2            3  c 

p table.raw_records
[[1, "a"], [2, "b"], [3, "c"]]
```

### Arrow::Table#raw_records の実装を紐解いていく

Ruby の C API `rb_define_method` を利用して、`raw_records` は定義されています。
メソッドの実態はどうやら `red_arrow::table_raw_records` で実装されていそうですね。

```c++
// ruby/red-arrow/ext/arrow/arrow.cpp
auto cArrowTable = rb_const_get_at(mArrow, rb_intern("Table"));
rb_define_method(cArrowTable, "raw_records",
                 reinterpret_cast<rb::RawMethod>(red_arrow::table_raw_records),
                 0);
```

ref: [https://github.com/apache/arrow/blob/main/ruby/red-arrow/ext/arrow/arrow.cpp#L86-L89](https://github.com/apache/arrow/blob/main/ruby/red-arrow/ext/arrow/arrow.cpp#L86-L89)

### red_arrow::table_raw_records

`red_arrow::table_raw_records` の実装は下記のようになっています。
今回は全てを読み解いていくのは大変なので主要な部分を Pick Up して読んでいきます。

```c++
// ruby/red-arrow/ext/arrow/raw-records.cpp
namespace red_arrow {
  VALUE
  table_raw_records(VALUE rb_table) {
    auto garrow_table = GARROW_TABLE(RVAL2GOBJ(rb_table));
    auto table = garrow_table_get_raw(garrow_table).get();
    const auto n_rows = table->num_rows();
    const auto n_columns = table->num_columns();
    auto records = rb_ary_new_capa(n_rows);

    try {
      RawRecordsBuilder builder(records, n_columns);
      builder.build(*table);
    } catch (rb::State& state) {
      state.jump();
    }

    return records;　
  }
}
```

ref: [https://github.com/apache/arrow/blob/main/ruby/red-arrow/ext/arrow/raw-records.cpp#L167-L183](https://github.com/apache/arrow/blob/main/ruby/red-arrow/ext/arrow/raw-records.cpp#L167-L183)

まず確認したいのがメソッドの戻り値の型です。これは `VALUE` 型になっています。
[実は C レベルで Ruby の値の型は全て VALUE 型で表現されます](https://docs.ruby-lang.org/en/3.0/extension_rdoc.html#label-Basic+Knowledge)。なので、ここでは戻り値も引数も `VALUE` 型として表現されています。 

```c++
VALUE
table_raw_records(VALUE rb_table) { }
```

次に確認したいのが下記の部分です。C++ では Ruby とは異なり配列の大きさを動的に変化させるにはコストがかかるので、決められるところでは事前に大きさを決めておくのが大切です。
`n_rows` の分だけ `rb_ary_new_capa` を利用して配列の大きさを確保しています。今回の例ですと `[[1, "a"], [2, "b"], [3, "c"]]` のような値が返るので `3` とかになりますね。

```c++
auto records = rb_ary_new_capa(n_rows);
```

最後にメソッドの中核となる部分ですね。
RawRecordsBuilder class のインスタンスを生成して `build` メソッドを呼んでいます。
次はここで実際になにをやっているのかを見ていきます。

```c++
RawRecordsBuilder builder(records, n_columns);
builder.build(*table);
```

その前に最後は `records` を返していますね。
メソッドの戻り値としての `[[1, "a"], [2, "b"], [3, "c"]]` を返している部分ですね。
```c++
return records;
```

### RawRecordsBuilder#build

`RawRecordsBuilder` の実装は下記のようになっています。
こちらも全てを読み解いていくのは大変なので主要な部分を Pick Up して読んでいきます。

```c++
// ruby/red-arrow/ext/arrow/raw-records.cpp
class RawRecordsBuilder : private Converter, public arrow::ArrayVisitor {
public:
  explicit RawRecordsBuilder(VALUE records, int n_columns)
    : Converter(),
      records_(records),
      n_columns_(n_columns),
      is_produce_mode_(false) {
  }

  void build(const arrow::Table& table) {
    rb::protect([&] {
      const auto n_rows = table.num_rows();
      for (int64_t i = 0; i < n_rows; ++i) {
        auto record = rb_ary_new_capa(n_columns_);
        rb_ary_push(records_, record);
      }
      for (int i = 0; i < n_columns_; ++i) {
        const auto& chunked_array = table.column(i).get();
        column_index_ = i;
        row_offset_ = 0;
        for (const auto array : chunked_array->chunks()) {
          check_status(array->Accept(this),
                       "[table][raw-records]");
          row_offset_ += array->length();
        }
      }
      return Qnil;
    });
  }
}
```

ref: [https://github.com/apache/arrow/blob/main/ruby/red-arrow/ext/arrow/raw-records.cpp#L50-L68](https://github.com/apache/arrow/blob/main/ruby/red-arrow/ext/arrow/raw-records.cpp#L50-L68)

こちらが、RawRecordsBuilder のコンストラクターになります。
特に複雑なことはやっておらず、引数で受け取った `records` と `n_columns` を、それぞれ `records_` と `n_columns_` に格納しています。
C++ には Ruby と違いインスタンス変数の概念がないので変数名の最後に `_` をつけるなどコーディングスタイルでの識別を行うことが多いようです。なので `records_` と `n_columns_` はインスタンス変数としてこれから扱っていきます。

```c++
explicit RawRecordsBuilder(VALUE records, int n_columns)
  : Converter(),
    records_(records),
    n_columns_(n_columns) {
}
```

次に、`RawRecordsBuilder#build` を見ていきます。
`build` メソッド内でも色々やっていそうですね。一つずつ見ていきます。

まず最初にやっているのは、行の数分だけ格納用の領域を確保するということをやっています。
`record` ローカル変数に `rb_ary_new_capa` を利用しカラムの数ぶんだけの領域を確保した配列を代入しています。
その `record` を `rb_ary_push` を通して `records_` に代入して行ってます。今回の例で言うと `[[], [], []]` のような構造が作られたイメージです。

```c++
void build(const arrow::Table& table) {
  rb::protect([&] {
    const auto n_rows = table.num_rows();
    for (int64_t i = 0; i < n_rows; ++i) {
      auto record = rb_ary_new_capa(n_columns_);
      rb_ary_push(records_, record);
    }
    // ...
  });
}
```

次は、先ほど作った構造に実際に値を代入をしていく部分になります。
見ていく前にここから少し Arrow 形式のデータ構造を元にしたコードになっています。なので今回全てを説明しませんが、気になる方は[こちらの動画](https://youtu.be/bpuJWC9_USY?t=273)を見てみてください。

`table.column(i).get()` では、表形式なデータ Arrow Table からカラム毎のデータを取得しています。
今回の例で言うと、`number: [1, 2, 3]` や `string: ['a', 'b', 'c']` に相当する部分になります。最初の loop では `number` のみの配列を処理し次の loop 
では `string` のみの配列を処理するみたいな形です。

```c++
for (int i = 0; i < n_columns_; ++i) {
  const auto& chunked_array = table.column(i).get();
  column_index_ = i;
  row_offset_ = 0;
  // ...
}
```

そして、渡された配列毎に `check_status(array->Accept(this), "[table][raw-records]")` の処理を呼んでいます。実際にどんなことをしているのか見ていきましょう。
- 具体的に渡される配列は、`[1, 2, 3]` みたいな構造が渡されるイメージです(厳密には違うのですが簡略化のため)

```c++
for (const auto array : chunked_array->chunks()) { // ここの構造は今回説明しませんが、カラム毎の配列が渡されるイメージでいてください
  check_status(array->Accept(this),
                "[table][raw-records]");
  row_offset_ += array->length();
}
```

今回の大事な部分が `array->Accept(this)` になります。ここでは Accept に渡された `VISIT` メソッドを呼ぶようになっています。
では、`RawRecordsBuilder` で定義されている `VISIT` メソッドを実際に見てみましょう。
下記で少しだけ詳しくみると下記のようになっていますが、今回は説明を一旦省略します。

```c++
// cpp/src/arrow/array/array_base.cc
Status Array::Accept(ArrayVisitor* visitor) const {
  return VisitArrayInline(*this, visitor);
}
```
```c++
// cpp/src/arrow/visit_array_inline.h
template <typename VISITOR, typename... ARGS>
inline Status VisitArrayInline(const Array& array, VISITOR* visitor, ARGS&&... args) {
  switch (array.type_id()) {
    ARROW_GENERATE_FOR_ALL_TYPES(ARRAY_VISIT_INLINE);
    default:
      break;
  }
  return Status::NotImplemented("Type not implemented");
}
```
```c++
// cpp/src/arrow/visit_array_inline.h
#define ARRAY_VISIT_INLINE(TYPE_CLASS)                                                  
  case TYPE_CLASS##Type::type_id:                                                       
    return visitor->Visit(                                                              
        internal::checked_cast<const typename TypeTraits<TYPE_CLASS##Type>::ArrayType&>(
            array),                                                                     
        std::forward<ARGS>(args)...);
```

### RawRecordsBuilder::VISIT(この表記は正確ではないですmm)

ここでは、渡された配列の型ごとに適切な関数を呼ぶような形で作られています。いわゆる Visitor パターンというやつですね。（[Visitor パターンの説明はここでされてます。](https://youtu.be/6Zb_jEKKznk?list=PLKb0MEIU7gvTj2_vQJMUK6HUu_R6V9oX5&t=155)）
そして実際に visit されて実行されるのは、`convert(array)` 関数になります。実はこれが今回の最も重要な部分になります。みていきましょう！
```c++
// ruby/red-arrow/ext/arrow/raw-records.cpp
#define VISIT(TYPE)                                                     \
      arrow::Status Visit(const arrow::TYPE ## Array& array) override { \
        convert(array);                                                 \
        return arrow::Status::OK();                                     \
      }
```

ref: [https://github.com/apache/arrow/blob/main/ruby/red-arrow/ext/arrow/raw-records.cpp#L71-L75](https://github.com/apache/arrow/blob/main/ruby/red-arrow/ext/arrow/raw-records.cpp#L71-L75)

### RawRecordsBuilder convert

`convert` メソッド内では大きく二つの分岐があります。渡された配列に null となる値があるか否かです。
今回は、よりシンプルな null が存在しない場合の処理（ else の部分）を追っていきます。
```c++
// ruby/red-arrow/ext/arrow/raw-records.cpp
template <typename ArrayType>
void convert(const ArrayType& array) {
  const auto n = array.length();
  if (array.null_count() > 0) {
    for (int64_t i = 0, ii = row_offset_; i < n; ++i, ++ii) {
      auto value = Qnil;
      if (!array.IsNull(i)) {
        value = convert_value(array, i);
      }
      auto record = rb_ary_entry(records_, ii);
      rb_ary_store(record, column_index_, value);
    }
  } else {
    for (int64_t i = 0, ii = row_offset_; i < n; ++i, ++ii) {
      auto record = rb_ary_entry(records_, ii);
      rb_ary_store(record, column_index_, convert_value(array, i));
    }
  }
}
```

ref: [https://github.com/apache/arrow/blob/main/ruby/red-arrow/ext/arrow/raw-records.cpp#L115-L133](https://github.com/apache/arrow/blob/main/ruby/red-arrow/ext/arrow/raw-records.cpp#L115-L133)

渡された配列の長さの分だけ下記の処理を繰り返します。
今回の例で言うと `[1, 2, 3]` が渡された場合 3 回ループを回すイメージですね。
次にループ内の処理を見ていきます。

```c++
const auto n = array.length();
if (array.null_count() > 0) {
  // ...
} else {
  for (int64_t i = 0, ii = row_offset_; i < n; ++i, ++ii) {
    // ...
  }
}
```

まず最初に `rb_ary_entry` を利用して先ほど行の分だけ空の配列を入れた `records_` から、指定した行の空配列を取得してきます。（Ruby で言う array[0] みたいなイメージですね。）
取得した配列を record に格納します。次に、`rb_ary_store` を利用して `column_index_` で指定した行に `convert_value` で変換した値を代入します。(Ruby で言う array[0] = 1 みたいなイメージですね。)
そうすることで列単位で渡されたデータを行単位のデータに追加していくことができます。

```c++
auto record = rb_ary_entry(records_, ii);
rb_ary_store(record, column_index_, convert_value(array, i));
```

具体的な例としては下記のようなステップで列ごとに対象の行のデータに追加していく感じです。
こうすることで、列データごとに処理をまわし結果として行データに変換できるようにしています。
```
1. 1列目のデータ: number: [1, 2, 3]
2. 全ての行のデータ: [[1], [2], [3]]
3. 2列目のデータ: string: ['a', 'b', 'c']
4. 全ての行のデータ: [[1, 'a'], [2, 'b'], [3, 'c']]
```

最終的には、変換が終わった `records_` つまり `records` が結果として返るわけです。
これが、`Arrow::Table#raw_records` の内部実装になります。
ここからわかることは、行単位で結果を逐次的に返すためには、既存の列ごとに行っている処理を行ごとに実行する実装をしてあげると良さそうですね。
次回は実際に手を動かして実装していきましょう！
