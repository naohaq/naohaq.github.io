---
layout: post
title:  "ポインタを使ったデータ構造をHaskellの代数的データ型と相互変換したい"
date:   2019-01-28 20:15:40 +0900
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

さて、このようなデータをHaskellで扱いたくなったときのことを考えます。上記のようなデータを受け取ったり返したりするCの関数をFFIで呼びたくなったとか、そういうCの関数を [QuickCheck](http://hackage.haskell.org/package/QuickCheck) でテストしたくなったとか。

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

ひとまず同じ構造のデータ型は定義できましたが、このままではHaskell的には使い勝手がよくありません。 `Ptr a` 型の値の操作は基本的にIOアクションになるので、関数的な記述の中に組込むのには難があります。例えば、 QuickCheck の [Arbitrary](http://hackage.haskell.org/package/QuickCheck/docs/Test-QuickCheck-Arbitrary.html) を使ったテストパターンの生成ができません。

しかたないので、 `Ptr` を使う代わりに `Maybe` を使って、 `Vec3DList` とisomorphicなデータ型を定義してやりましょう。名前は `HVec3DList` とでもしておきます。

```haskell
data HVec3DList =
  HVec3DList
  { hx :: Double
  , hy :: Double
  , hz :: Double
  , hnext :: Maybe HVec3DList
  }
```

`NULL` ポインタをリストの終端に使う代わりに、 `Nothing` でリストの終端を表現してやるわけです。

### 相互変換するでござる

Isomorphicなデータ型をできたところで、相互に変換できなくては意味がありません。

それに当たって、ポインタの先からデータを読んだりポインタの先にデータを書いたりする必要があります。

ポインタの先のデータを読み書きするのには [Storable](http://hackage.haskell.org/package/base-4.12.0.0/docs/Foreign-Storable.html#t:Storable) クラスの `peek` / `poke` メソッドを使うのですが、先程定義した `Vec3DList` はまだ `Storable` のインスタンスにはなっていません。`Vec3DList` を `Storable` のインスタンスにするには、より細かいメモリ操作を使って `peek` や `poke` を定義してやらないといけません。なんだかつらみのある世界になってきました。

ですが、今のGHCには[Generics](https://wiki.haskell.org/GHC.Generics)という強力な仕組みがあり、そこを自動でやらせることができます。([derive-storable](http://hackage.haskell.org/package/derive-storable) パッケージが必要)

```haskell
{-# LANGUAGE DeriveGeneric #-}
{-# LANGUAGE StandaloneDeriving #-}
{-# LANGUAGE FlexibleInstances #-}

module Vec3DList where

import GHC.Generics (Generic)
import Foreign.Ptr (Ptr, nullPtr)

import Foreign.Storable
import Foreign.Storable.Generic
import Foreign.Marshal.Alloc

data CVec3DList =
  CVec3DList
  { x :: Double
  , y :: Double
  , z :: Double
  , next :: Ptr CVec3DList
  } deriving Generic

data HVec3DList =
  HVec3DList
  { hx :: Double
  , hy :: Double
  , hz :: Double
  , hnext :: Maybe HVec3DList
  }

deriving instance Show CVec3DList

instance GStorable CVec3DList
```

分かりにくくなりそうだったので、C側のデータ型は `CVec3DList` という名前に変更しました。

データ型の定義に `deriving Generic` 宣言を追加することで `CVec3DList` 型に Generics が使えるようになり、`CVec3DList` 型を [GStorable](http://hackage.haskell.org/package/derive-storable-0.1.2.0/docs/Foreign-Storable-Generic.html#t:GStorable) のインスタンスとすることで、 Generics を使って `Storable` に必要なメソッドが自動的に導出されます。

では導出されたメソッドたちを使って、Haskell側のデータ型との相互変換を記述しましょう。

```haskell
toCVec3DList :: HVec3DList -> IO (Ptr CVec3DList)
toCVec3DList (HVec3DList x' y' z' r) = do
  r' <- case r of
          Just v  -> toCVec3DList v
          Nothing -> return nullPtr
  p <- malloc :: IO (Ptr CVec3DList)
  poke p (CVec3DList x' y' z' r')
  return p

fromCVec3DList :: Ptr CVec3DList -> IO (Maybe HVec3DList)
fromCVec3DList p
  | p == nullPtr  = return Nothing
  | otherwise     = do
      (CVec3DList x' y' z' r) <- peek p
      r' <- fromCVec3DList r
      return (Just $ HVec3DList x' y' z' r')
```

### 型定義を高カインド多相でまとめる

ところで、C側のデータ型とHaskell側のデータ型をそれぞれ `CVec3DList` 、 `HVec3DList` と定義していたわけですが、どうにも冗長です。しかも、レコード名の重複を防ぐために見苦しい感じの定義になってしまっています。

そもそも、 `CVec3DList` と `HVec3DList` とでは
```haskell
next :: Ptr CVec3DList
```
```haskell
hnext :: Maybe HVec3DList
```
のところが `Ptr` か `Maybe` かという違いしかないので、なんとかまとめたいところです。

そこで、こいつらを **高カインド多相** (higher kind polymorphism)を使って多相型にしてしまいましょう。要は、 `Ptr` や `Maybe` が入るところを型変数にしてやればいいわけです。この場合、ここに入るのは種 `* -> *` の型コンストラクタになります。

以下がその定義です。
```haskell
data Vec3DList f =
  Vec3DList
  { x :: Double
  , y :: Double
  , z :: Double
  , next :: f (Vec3DList f)
  } deriving Generic
```
`Vec3DList` は種 `* -> *` の型コンストラクタ1つを引数にとる型コンストラクタで、`next` の型はなんだか不動点演算子みたいになりました。今までの `CVec3DList` と `HVec3DList` は以下のような型シノニムになります。
```haskell
type HVec3DList = Vec3DList Maybe
type CVec3DList = Vec3DList Ptr
```

### コード例

以上のテクニックを使ったコードの例をgistの [Vec3DList.hs](https://gist.github.com/naohaq/0a39e8614d288b562c47a2f67e781d2d) にあげておきます。
