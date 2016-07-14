---
title: Applicative
format: latex
...


Definition
==========

```hs
class Functor f => Applicative f where
    pure :: a -> f a
    infixl 4 <*>
    (<*>) :: f (a -> b) -> f a -> f b
```


Laws
====

### 1. Identity

```
pure id <*> v = v
```

### 2. Homomorphism

```
pure f <*> pure x = pure (f x)
```

### 3. Interchange

```
u <*> pure y = pure ($ y) <*> u
```

### 4. Composition

```
u <*> (v <*> w) = pure (.) <*> u <*> v <*> w
```

Recall:

a)  `pure :: a -> f a`
b)  `(<*>) :: f (a -> b) -> f a -> f b`
c)  `(.) :: (b -> c) -> (a -> b) -> a -> c`

From the above,

```hs
pure (.) :: f ((b -> c) -> (a -> b) -> a -> c)
u :: f (b -> c)
v :: f (a -> b)
w :: f a
```

We can verify that these types for `u`, `v`, and `w` satisfy the 4th law above.

### Additional Law: Applicative and Functor Relationship

```hs
fmap g x = pure g <*> x
```

or

```hs
g <$> x = pure g <*> x
```

where `<$>` is a synonym for `fmap`.

Exercises
---------

1. One might imagine a variant of the interchange law that says something about
applying a pure function to an effectful argument. Using the above laws, prove
that:

```hs
pure f <*> x = pure (flip ($)) <*> x <*> pure f
```

--

```hs
pure f <*> x = (<*>) (pure f) x
             = flip (<*>) x (pure f)
```

```hs
pure f <*> x = pure f <*> (pure id <*> x)
             = pure (.) <*> pure f <*> pure id <*> x
             = ((pure (.) <*> pure f) <*> pure id) <*> x

pure (.) :: f ((b -> c) -> (a -> b) -> a -> c)
```

Notes:

```hs
f :: a -> b
x :: f a

($) :: (a -> b) -> a -> b
flip ($) :: a -> (a -> b) -> b
pure (flip ($)) :: f (a -> (a -> b) -> b)

(<*>) :: f (a -> b) -> f a -> f b
flip (<*>) :: f a -> f (a -> b) -> f b

flip <*> pure ($) = pure (flip ($))

pure f <*> x = pure (($) f) <*> x -- legit?
             = pure ($) <*> pure f <*> x

pure ($ y) <*> u = pure (flip ($) y) <*> u
                 = pure (flip ($)) <*> pure y <*> u
```

Pfft, scrap this for now, come back later.

```hs
pure f <*> x = pure (flip ($)) <*> x <*> pure f
             = (pure (flip ($)) <*> x) <*> pure f                    --- (<*>) associates left
             = pure ($ f) <*> (pure (flip ($)) <*> x)                --- interchange
             = ((pure (.) <*> pure ($ f)) <*> pure (flip ($))) <*> x --- composition
             = pure ((.) ($ f) (flip ($))) <*> x                     --- homomorphism (x2)
             = pure (\a -> ($ f) (flip ($) a)) <*> x                 --- defn (.)
             = pure (\a -> (flip ($) a) f) <*> x                     --- defn ($)
             = pure (\a -> f $ a) <*> x                              --- defn flip
             = pure (\a -> f a) <*> x                                --- defn ($)
             = pure f <*> x                                          --- eta-reduction
```


Instances
=========

We can implement the list type constructor `[]` as an instance of `Applicative`
in two ways:

1. As an ordered collection of elements (we'll call this a `ZipList`).
2. As contexts representing multiple results of a nondeterministic computation
   (we'll call this a list, with constructor `[]`).

For the first, we define

```hs
newtype ZipList a = ZipList { getZipList :: [a] }

instance Applicative ZipList where
    pure = undefined -- see Exercise 2 below
    (ZipList gs) <*> (ZipList xs) = ZipList (zipWith ($) gs xs)
```

***Aside:***

*The RHS of the above `newtype` line uses 'named field' or 'record' syntax. In
this case, the RHS creates a type constructor `ZipList [a]` and accessor
function `getZipList` with type signature*

```hs
getZipList :: ZipList a -> [a]
```

*This will return the regular list contained 'inside' of a `ZipList`.*

For the second, we have

```hs
instance Applicative [] where
    pure x = [x]
    gs <*> xs = [ g x | g <- gs, x <- xs]
```

Exercises
---------

1. Implement an instance of `Applicative` for `Maybe`.

--

**My answer:**

```hs
instance Applicative Maybe where
    pure = Just
    (Just g) <*> (Just x) = Just (g x)
    _ <*> _ = Nothing
```

**Implementation in GHC.Base:**

```hs
instance Applicative Maybe where
    pure = Just
    Just f <*> m = fmap f m
    Nothing <*> _m = Nothing
```

Both should work, the latter leverages the `Functor` instance of `Maybe` and the
fact that `pure` and `Just` are the same (i.e. `Just` wraps an expression in an
effect-free context).

---

2. Determine the correct definition of `pure` for the `ZipList` instance of
   `Applicative` -- there is only one implementation that satisfies the law
   relating `pure` and `(<*>)`.

--

For `ZipList`, `pure :: a -> ZipList a`. If we let

```hs
pure x = ZipList [x]
```

Then we find that this breaks the [first Applicative law](#1. Identity). Notice
that:

```
pure (0+) <*> ZipList [1,2,3]
= ZipList [(0+)] <*> ZipList [1,2,3]
= ZipList (zipWith ($) [(0+)] [1,2,3])
= ZipList [1]
```

We can see that this fails with infinite lists on the RHS as well, since the
result of `zipWith` is only as long as the shorter of the two input lists. To
correct this, we have the following instance of `Applicative` for `ZipList`:

```hs
instance Applicative ZipList where
    pure x = ZipList (repeat x)
    (ZipList gs) <*> (ZipList xs) = ZipList (zipWith ($) gs xs)
```

`repeat x` creates an infinite list of `x`'s (same as `[x,x..]`). This ensures
that the result of `zipWith` is always as long as the shorter (finite) list and
no elements are truncated.

Intuition
=========

Alternative Formulation
=======================

Equivalent formulation of `Applicative`:

```hs
class Functor f => Monoidal f where
    unit :: f ()
    (**) :: f a -> f b -> f (a,b)
```

Laws
----

***Note: The `~=` symbols below refer to 'isomorphisms' as opposed to
equality.***

### Left Identity

```
unit ** v ~= v
```

### Right Identity

```
u ** unit ~= u
```

### Associativity

```
u ** (v ** w) ~= (u ** v) ** w
```

### Naturality

(This is a free theorem in Haskell.)

```
fmap (g ** h) (u ** v) = fmap g u ** fmap h v
```

Exercises
---------

1. Implement `pure` and `(<*>)` in terms of `unit` and `(**)`, and vice versa.

--

---

2.

--

---

3.

