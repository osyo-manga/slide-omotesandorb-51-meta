### 表参道.rb #51
- - -

## メタプログラミング
## と
## SAD STORY

---

#### 自己紹介
- - -

* なまえ  : おしょー
* Twitter : [@pink_bangbi](https://twitter.com/pink_bangbi)
* github  : [osyo-manga](https://github.com/osyo-manga)
* ブログ  : [Secret Garden(Instrumental)](http://secret-garden.hatenablog.com)
* Rails 歴 1年半
* 趣味で Ruby にパッチを投げたりしてます
* 最近のトレンド
  * `binding_of_caller`

---

## 今日話すこと

---

## メタプログラミング
## と
## SAD STORY

---

## --- お題 ---
### Hash の添字アクセスを
### メソッド呼び出しにしよう

---

### Hash の添字アクセス
- - -

* 通常は #[] や #[]= を使って要素を参照する

```ruby
homu = { id: 1, name: :homu, age: 14}
homu[:id]     # => 1
homu[:name]   # => :homu
homu[:age]    # => 14

homu[:age] = 15
homu[:category] = "human"

pp homu
# => {:id=>1, :name=>:homu, :age=>15, :category=>"human"}


users = [{ name: :homu }, { name: :mami }, { name: :mado }]
pp users.map { |user| user[:name] }
# => [:homu, :mami, :mado]
```

---

### メソッド呼び出しでアクセス
- - -

* #[] ではなくてメソッド呼び出しでアクセスする

```ruby
homu = { id: 1, name: :homu, age: 14}
pp homu.id       # => 1
pp homu.name     # => :homu
pp homu.age      # => 14

homu.age = 15
homu.category = "human"

pp homu
# => {:id=>1, :name=>:homu, :age=>15, :category=>"human"}


users = [{ name: :homu }, { name: :mami }, { name: :mado }]
pp users.map { user.name }
# => [:homu, :mami, :mado]
```

---

## どうやって実現する？ 🤔

---

## method_missing を使う！

---

#### method_missing とは
- - -

* メソッド呼び出しに失敗した時に呼び出される

```ruby
class X
  # インスタンスメソッドとして定義する
  # name は呼び出されたメソッドの名前
  # *args は呼び出されたメソッドの引数
  def method_missing(name, *args)
    { name: name, args: args }
  end
end

x = X.new

# X#hoge というメソッドは存在しないので
# method_missing に委譲される
pp x.hoge 1, "homu", 14
# => {:name=>:hoge, :args=>[1, "homu", 14]}

# 定義済みメソッドの場合はそのまま
pp x.class
# => X
```

---

## Hash に生やしたろ！！

---

```ruby
class Hash
  # MEMO: xxx.age = 14 と呼び出すと
  #       method_missing(:age=, 14) となる
  def method_missing(name, *args)
    if name =~ /(.*)=$/
      self[$1.to_sym] = args[0]
    else
      super unless has_key?(name)
      self[name]
    end
  end
end

homu = { id: 1, name: :homu, age: 14}

# id や name というメソッドが存在しないので method_missing に委譲される
pp homu.id       # => 1
pp homu.name     # => :homu
pp homu.age      # => 14

homu.age = 15
homu.category = "human"

pp homu
# => {:id=>1, :name=>:homu, :age=>15, :category=>"human"}
```

---

### と、言う感じで
### このように動的に処理を行うことを
### メタプログラミングといいます

---

### method_missing 以外にも
- - -

* const_missing
* define_method
* respond_to?
* send, public_send
* instance_exec, instance_eval
* instance_variable_set, instance_variable_get
* Class.new, Module.new
* class_eval, module_eval
* etc...

---

## ここからが SAD STORY

---

## 皆さんは Refinements という
## 機能を知っていますか？

---

#### クラス拡張
- - -

* クラス拡張を行うと全てのコードに対してその拡張が反映されてしまう
* これはしばしばよからぬ副作用をもたらす

```ruby
class String
  # すべての String に #twice が定義される
  def twice
    self + self
  end

  # 既存のメソッドを書き換えることも出来る
  def to_s
    twice
  end
end

p "homu".twice    # => "homuhomu"
p "mami".to_s     # => "maimami"
```

---

#### Refinements を使うと
- - -

* クラス拡張を適用する範囲を制限する

```ruby
module StringEx
  refine String do
    def twice
      self + self
    end
  end
end

class X
  # このクラス内でのみクラス拡張が反映される
  using StringEx
  def hoge
    p "homu".twice    # => "homuhomu"
  end
end

# Error: undefined method
"mami".twice
```

---

## method_missing も
## Refinements で定義しよう

---

```ruby
module HashEx
  refine Hash do
    def method_missing(name, *args)
      if name =~ /(.*)=$/
        self[$1.to_sym] = args[0]
      else
        super unless has_key?(name)
        self[name]
      end
    end
  end
end

using HashEx

homu = { id: 1, name: :homu, age: 14}

# Error: undefined method
pp homu.name
```


---

## なぜか動かない？🤔

---

## 答え

---

## Refinements は method_missing 
## に対応してないから！！！！

---

## SAD STORY
- - -

* Refinements で method_missing を定義しても反映されない
* これは Ruby の言語仕様がそうなっているから
* [#15374](https://bugs.ruby-lang.org/issues/15374) で method_missing が反映されるように提案した
* しかし、残念ながら Reject されてしまった… 😇

---

## そもそも…

---

* 昔々[同様の事を行う gem](https://github.com/osyo-manga/gem-hash_with_key_access_method) をつくっていた
* メタプログラミングネタを探している時にこれの存在を思い出した
* このスライドをつくりながら実装を見ていたら method_missing に対応しているようにみえた
* README やテストも method_missing が呼び出されることを期待しているようになっていた
* 🤔

---

## 調べてみると…

---

## Ruby 2.2 系では
## method_missing の呼び出しが許されてた
## ナンテコッタ／(^o^)＼

---

## つまり Ruby 2.2 から 2.3 で非互換な変更があった
## 流石に Ruby 2.2 は古すぎるやろ…

---

#### まとめ
- - -

* Ruby のメタプログラミングはとても強力、メタプログラミング楽しい
* ただし、用法用量を守って正しく使わないと思わぬ副作用があるので注意
* Refinements 無限につれえ

---


# 宣伝

---

### Ruby Hack Challenge Holiday
- - -

* [Ruby Hack Challenge Holiday](https://rhc.connpass.com/event/145802/) というイベントをやっています
* Ruby 本体を自分でビルドしたりハックしたりするイベント           <!-- .element: class="fragment" -->
* Ruby のコミッタの方が主催しているので Ruby について聞けるチャンス       <!-- .element: class="fragment" -->
* Ruby 本体の開発に興味がある人はぜひ！！       <!-- .element: class="fragment" -->
* 次回は 10/06(日) 開催予定       <!-- .element: class="fragment" -->

---

## ご清聴
## ありがとうございました
