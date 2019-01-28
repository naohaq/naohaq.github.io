---
layout: post
title:  "ポインタを使ったデータ構造をHaskellの代数的データ型と相互変換したい"
date:   2019-01-26 17:27:14 +0900
categories: haskell
---
### 再帰的なデータ構造

Cのような言語で再帰的なデータ構造(連結リストなど)を扱う場合、ポインタを使って

```c
struct ST_VEC3D_LIST {
    double x;
    double y;
    double z;
    struct ST_VEC3D_LIST * next;
};
```

というような構造体を定義してやるのが常套手段です。

要素を追加するときは `next` ポインタでつないでいって、リストを手繰るときは `next` ポインタを再帰的に参照していくことになります。リストの終端は `next` に `NULL` を代入することで表現します。

さて、このようなデータをHaskellで扱いたくなったときのことを考えます。上記のようなデータを受け取ったり返したりするCの関数をFFIで呼びたくなったとか、そういうCの関数を[QuickCheck](http://hackage.haskell.org/package/QuickCheck)でテストしたくなったとか。

とりあえずHaskellに用意されている `Foreign.Ptr` を使って、`struct ST_VEC3D_LIST` と同じ構造を定義してみましょう。

```haskell
import Foreign.Ptr

data Vec3DList =
  Vec3DList
  { x :: Double
  , y :: Double
  , z :: Double
  , next :: Ptr Vec3DList
  }
```

ひとまず同じ構造のデータ型は定義できましたが、このままではHaskell的には使い勝手がよくありません。 `Ptr a` 型の値の操作は基本的にIOアクションになるので、関数的な記述の中に組込むのには難があります。例えば、QuickCheckの[Arbitrary](http://hackage.haskell.org/package/QuickCheck/docs/Test-QuickCheck-Arbitrary.html)を使ったテストパターンの生成ができません。

しかたないので、 `Ptr` を使う代わりに `Maybe` を使って、 `Vec3DList` とisomorphicなデータ型を定義してやりましょう。名前は `HVec3DList` とでもしておきます。

```haskell
data HVec3DList =
  HVec3DList
  { x :: Double
  , y :: Double
  , z :: Double
  , next :: Maybe HVec3DList
  }
```

`NULL` ポインタをリストの終端に使う代わりに、 `Nothing` でリストの終端を表現してやるわけです。
