---
layout: post
title:  "ポインタを使ったデータ構造をHaskellの代数的データ型と相互変換したい"
date:   2019-01-28 16:49:58 +0900
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

### 相互変換するでござる

Isomorphicなデータ型をできたところで、相互に変換できなくては意味がありません。

それに当たって、ポインタの先からデータを読んだりポインタの先にデータを書いたりする必要があります。

ポインタの先のデータを読み書きするのには[Storable](http://hackage.haskell.org/package/base-4.12.0.0/docs/Foreign-Storable.html#t:Storable)クラスの `peek` / `poke` メソッドを使うのですが、先程定義した `Vec3DList` はまだ `Storable` のインスタンスにはなっていません。`Vec3DList` を `Storable` のインスタンスにするには、より細かいメモリ操作を使って `peek` や `poke` を定義してやらないといけません。なんだかつらみのある世界になってきました。

ですが、今のGHCには[Generics](https://wiki.haskell.org/GHC.Generics)という強力な仕組みがあり、そこを自動でやらせることができます。

```haskell
{-# LANGUAGE DeriveGeneric #-}
{-# LANGUAGE StandaloneDeriving #-}
{-# LANGUAGE FlexibleInstances #-}

module Vec3DList where

import GHC.Generics (Generic)

import Foreign.Storable
import Foreign.Storable.Generic

data Vec3DList =
  Vec3DList
  { x :: Double
  , y :: Double
  , z :: Double
  , next :: Ptr Vec3DList
  } deriving Generic

deriving instance Show Vec3DList

instance GStorable Vec3DList
```



