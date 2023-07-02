---
layout: post
title: "Apache Arrow の Ruby バインディングに新機能を追加する 実装編(1/3)"
date: 2023-06-30 09:18:32 +0900
categories: Red-Data-Tools
---

## 最初に
[Red Data Tools開発者に聞け！第13回 Apache Arrow の Ruby バインディングに新機能を追加するシリーズ](https://www.youtube.com/watch?v=eZGLDFr4s90) の中で、kou さんが解説してくださった内容を簡単にまとめるシリーズになります。
今回は Part7 で紹介された内容をまとめていきます。
- 動画で kou さんがサクッと解説してくださっています。動画で見たい方は上のリンクを参照してください！

## 前回までのあらすじ
Apache Arrow の Ruby バインディングが手元で動作するようになったので、遂に Ruby バインディングに新機能を追加するフェーズまでやってきました！
では、実際にどんな機能を追加したく、どのように進めていくのかを確認していきます。

## どんな機能を追加したいの？
`Arrow::Table#raw_records`という、Arrow::Table オブジェクトから作成した表データを Ruby の配列形式に変換するメソッドがあります。今回は、この配列への変換をイテラブルにする`Arrow::Table#raw_each_record`という機能を追加したいです。
```console
> require 'arrow'
> arrow = Arrow::Table.new(number: [1, 2, 3], string: ['a', 'b', 'c'])
=> 
#<Arrow::Table:0x10a9d8ba8 ptr=0x12da8bd80>                                     
        number  string                                                          
0            1  a                                                               
1            2  b                                                               
2            3  c

> arrow.raw_records
=> [[1, "a"], [2, "b"], [3, "c"]]
```

### どんな時に嬉しいのか？
どんな時にイテラブルに処理すると嬉しいのかという疑問があると思います。
今回の例のように、データ量が少ない場合にはあまり問題になりません。しかし、不断 Arrow が取り扱うデータは大量であり、全てを一度に配列に変換すると処理が終わるまでに時間が掛かるという課題が存在します。

具体例として、ニューヨーク市の1ヶ月分のタクシー利用データ（100万件以上のデータ）を考えてみましょう。全体のデータを分析する必要はなく、最初の1000件だけを確認したいとします。全データの変換を待つのは時間がかかりますが、イテラブルな変換を利用すれば、最初の1000件のデータだけを短時間で取り出すことが可能になります。
- ref: [Added New York city's Taxi and Limousine Commission For Hire Vehicle trip support](https://github.com/red-data-tools/red-datasets-parquet/pull/11)

つまり、この機能があると嬉しい時は、「大量のデータの中から一部だけを効率よく分析したい」時となります。

## どのように進めていくのか
実際になぜ必要なのかを共有できたので、実際に開発していく方針を考えてみましょう！
今回は、初めての Apache Arrow の Ruby バインディングへの機能追加なので下記の手順で行なっていきます。

1. 追加したい機能を Ruby で実装してイメージを掴む <- **今回の記事はここまで**
2. Ruby が提供する C の API を学びながら追加対象に関連するコードを理解する
3. 追加機能の実装を行う

### 1. 追加したい機能を Ruby で実装してイメージを掴む

`Arrow::Table#raw_records`をイテラブルに実行する`Arrow::Table#raw_each_record`を Ruby で簡単に実装してみます。

```ruby
# /arrow/ruby/red-arrow/sample-each.rb
require 'arrow'

class Arrow::Table
  def each_raw_record
    raw_records.each do |row|
      yield row
    end
  end
end

Arrow::Table.new(number: [1, 2, 3], string: ['a', 'b', 'c']).each_raw_record { |row| p row }
```
```console
% bundle exec env DYLD_LIBRARY_PATH=$DYLD_LIBRARY_PATH:/tmp/local/lib ruby -I ext/arrow sample-each.rb
[1, "a"]
[2, "b"]
[3, "c"]
```
上記のように Ruby でイテレータを作成するのは `yield` を利用することで簡単に作成することができます。
今実装した例を元に、Ruby バインディングに機能追加をしてあげたら良さそうです。

大雑把には下記のフローで機能追加を実現しようと考えています。
1. 既存の `Arrow::Table#raw_records` を利用して、変換が終わり次第一つずつ `yield` してあげる形で、`Arrow::Table#raw_each_record` を実装する。
2. 次に既存の `Arrow::Table#raw_records` は一度に全てを変換してしまうので、一つずつ変換する機能を追加する。

次の記事では、今回実装イメージがついたので、実際に機能を追加する部分のコードを理解をするところから始めます。
