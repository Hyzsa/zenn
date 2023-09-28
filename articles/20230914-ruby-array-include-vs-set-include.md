---
title: "【Ruby】Array#include?はto_setで速くなる"
emoji: "💎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ruby"]
published: true
published_at: 2023-09-14 08:00
---

## はじめに

配列に特定の要素が含まれているかを判定したいとき、よく`Array#include?`を利用すると思いますが、その際に一工夫（`to_set`）してあげるだけで、より速く判定することができます。

## 速くなる理由

`Array#include?`は検索アルゴリズムの線型探索にあたり、データ量(=要素数)が多くなるほど処理に時間がかかります。
一方で`Set#include?`は検索アルゴリズムのハッシュ法にあたり、処理時間はデータ量(=要素数)に関係なく一定になるので、to_set した方が高速に判定することができるのです。

### Array#include?

配列の先頭から順番に、引数で渡された値と等しいかをチェックしていき、等しければ true を返します。

```ruby
[1, 2, 3, 4, 5]
# このような配列から4が含まれてるかを判定したい時
# ・先頭の1を見る。
# ・1は4ではないので、次の要素2を見る。
# ・2は4ではないので、次の要素3を見る。
# ・：以降同じ判定を繰り返し実施していき、4を見つけ次第 true を返す。
# ・※配列の要素全てを確認しても見つけられなかった場合は false を返す。
```

:::details Array#include?の実装を詳しく見てみる
https://github.com/ruby/ruby/blob/a98209b8a70345714ac5f3028e0591f3ee50bba7/array.c#L5212-L5225

C 言語で書かれた実装を見てみると、配列の先頭から順番に判定していることがわかりますね。
判定する要素が先頭から離れた位置にあればあるほど、一致するまでに時間がかかることが想像できます。
:::

### Set#include?

配列を渡して Set オブジェクトを生成した時、この Set オブジェクトは`{ 要素 => true, ... }`のようなハッシュを内部的に保持します。

```ruby
set = [1, 2, 3, 4, 5].to_set
# このときsetオブジェクトは以下のようなハッシュを保持する。
@hash = { 1 => true, 2 => true, 3 => true, 4 => true, 5 => true }
```

:::details to_set した時の実装を詳しく見てみる
【前提】`[1, 2, 3].to_set`を実行した状況を元に見ていきます。

### ①Enumerable#to_set

https://github.com/ruby/ruby/blob/4655d2108ef14e66f64496f9029f65ba2302d9ea/prelude.rb#L28-L30

単純に`Set.new([1, 2, 3])`しているだけですね。
new はオブジェクトを生成するメソッドなので、Set クラスのオブジェクトが生成されます。

### 　 ②Set#initialize

https://github.com/ruby/ruby/blob/a98209b8a70345714ac5f3028e0591f3ee50bba7/lib/set.rb#L243-L253

ここで注目してほしいのが、`@hash ||= Hash.new(false)`の部分です。
デフォルト値が false のハッシュを生成して、インスタンス変数の`@hash`に保持しています。
そして block は渡していないので、merge が実行されます。
initialize には new に与えた引数がそのまま渡されるので、enum には`[1, 2, 3]`が格納されてます。
そのため merge の引数として`[1, 2, 3]`が渡されます。

### ③Set#merge

https://github.com/ruby/ruby/blob/a98209b8a70345714ac5f3028e0591f3ee50bba7/lib/set.rb#L603-L613

可変長引数として受け取っているので enums は `[[1, 2, 3]]` という状態になります。
`enum`は配列になり`do_with_enum(enum) { |o| add(o) }`が実行されます。

### ④Set#do_with_enum

https://github.com/ruby/ruby/blob/a98209b8a70345714ac5f3028e0591f3ee50bba7/lib/set.rb#L272-L281

Array は each_entry を持っているので`enum.each_entry(&block) if block`が実行されます。
each_entry は、渡されたブロックを各要素に一度ずつ適用するので、`[1, 2, 3]`の各要素に対して add が実行されることになります。

#### ⑤Set#add

https://github.com/ruby/ruby/blob/a98209b8a70345714ac5f3028e0591f3ee50bba7/lib/set.rb#L519-L522

ここで各要素を`@hash`のキーとしてセットしています。
なので、`[1, 2, 3].to_set`を行うと内部的に以下のようなハッシュとして保持されるのです。

```ruby
@hash = { 1 => true, 2 => true, 3 => true }
```

:::

`Set#include?`では、このハッシュを参照することで、存在している時は true が返り、存在しない時は false が返ります。

```ruby
set = [1, 2, 3, 4, 5].to_set
set.include?(4)
# このときsetオブジェクトに内部的に保持されている以下のハッシュに対して @hash[4] で参照する。
# ・参照した結果 true が返る
@hash = { 1 => true, 2 => true, 3 => true, 4 => true, 5 => true }

set.include?(6)
# このときsetオブジェクトに内部的に保持されている以下のハッシュに対して @hash[6] で参照する。
# ・参照した結果存在しないのでfalseが返る
# ・※ nil ではなく false が返されるのは、以下ハッシュのデフォルト値として false が設定されているためです。
@hash = { 1 => true, 2 => true, 3 => true, 4 => true, 5 => true }
```

:::details Set#include?の実装を詳しく見てみる
https://github.com/ruby/ruby/blob/a98209b8a70345714ac5f3028e0591f3ee50bba7/lib/set.rb#L401-L403

見ての通り、Set オブジェクト生成時に内部的に保持したハッシュに対して、引数で指定された位置を参照しているだけです。
:::

このように、データ量(=要素数)がどれだけ増えても、常に指定された位置にアクセスしに行くので、高速で判定することができるのです。

## 実際に検証してみる

`Array#include?`を利用した時と`Set#include?`を利用した時の処理速度を比較して、どれくらいの差があるのかを確認してみます。

```ruby:検証用コード
def benchmark_include(length)
  array = (1..length).to_a
  set = array.to_set

  Benchmark.bm do |x|
    x.report('Array:') { array.include?(length) }
    x.report('Set:') { set.include?(length) }
  end
end

[100, 100_0, 100_00, 100_000, 100_000_0].each do |length|
  pp "length: #{length}"
  benchmark_include(length)
end
```

これを実行するとこのような結果になります。

```ruby:結果
"length: 100"
       user     system      total        real
Array:  0.000017   0.000000   0.000017 (  0.000003)
Set:  0.000003   0.000000   0.000003 (  0.000002)
"length: 1000"
       user     system      total        real
Array:  0.000006   0.000000   0.000006 (  0.000005)
Set:  0.000002   0.000000   0.000002 (  0.000002)
"length: 10000"
       user     system      total        real
Array:  0.000023   0.000012   0.000035 (  0.000035)
Set:  0.000002   0.000001   0.000003 (  0.000001)
"length: 100000"
       user     system      total        real
Array:  0.000232   0.000113   0.000345 (  0.000345)
Set:  0.000002   0.000001   0.000003 (  0.000002)
"length: 1000000"
       user     system      total        real
Array:  0.003023   0.000001   0.003024 (  0.003023)
Set:  0.000004   0.000001   0.000005 (  0.000003)
```

`Array#include?`では要素数が増えていくごとに処理に時間がかかっていますが、`Set#include?`では要素数が増えても処理速度が変わっていないことがわかりますね。

## まとめ

| メソッド       | 計算量(O 記法) | 特徴                                                                                |
| -------------- | -------------- | ----------------------------------------------------------------------------------- |
| Array#include? | O(n)           | ・検索アルゴリズムの線型探索にあたる <br>・データ量が多くなるほど処理に時間がかかる |
| Set#include?   | O(1)           | ・検索アルゴリズムのハッシュ法にあたる <br> ・処理時間はデータ量に関係なく一定      |

## 参考にさせていただいた記事

https://qiita.com/an_sony/items/708c47d073ad709431d6
https://nekomaho.hatenablog.jp/entry/2019/05/28/083000
