---
title: Functor
format: latex
...

Definition
==========

```hs
class Functor f where
    fmap :: (a -> b) -> f a -> f b
```


Instances
=========

The `Functor` instances for lists `[]` and `Maybe` are

```hs
instance Functor [] where
    fmap _ [] = []
    fmap g (x:xs) = g x : fmap g xs
    -- or we could just say fmap = map

instance Functor Maybe where
    fmap _ Nothing = Nothing
    fmap g (Just x) = Just (g x)
```

Exercises
---------

1. Implement `Functor` instances for `Either a` and `((->), e)`.

--

```hs
data Either a b = Left a | Right b
    deriving (Show)

instance Functor (Either e) where
    fmap _ (Left a) =  Left a
    fmap g (Right b) = Right (g b)

instance Functor (->) e where
    fmap g h = g . h
    -- or
    fmap = (.)
```

---

2. Implement `Functor` instances for `((,) e)` and for `Pair`, defined as
`data Pair a = Pair a a`.

--

```hs
instance Functor (,) e where
    fmap g (a, b) = (a, g b)

data Pair a = Pair a a

instance Functor Pair where
    fmap g (Pair a b) = Pair (g a) (g b)
```

---

3. Implement a `Functor` instance for the type `ITree`, defined as

```hs
data ITree a = Leaf (Int -> a)
             | Node [ITree a]
```

--

```hs
instance Functor ITree where
    fmap g (Leaf h) = Leaf (g . h)
    fmap g iTree = Node (map (fmap g) iTree)
```

---

4. Give an example of a type of kind `* -> *` which cannot be made into an
   instance of `Functor` (without using undefined).

--

```hs
data NotAFunctor a = NotAFunctor (a -> Int)
```

---

5. Is this statement true or false? *The composition of two `Functor`s is also a
   `Functor`.* If false, give a counterexample; if true, prove it by exhibiting
   some appropriate Haskell code.

--

True. Consider `Just (Just 1))` with the function (fmap (1+)). Recall the
`Functor` instance for `Maybe`:

```hs
instance Functor Maybe where
    fmap _ Nothing  = Nothing
    fmap g (Just x) = Just (g x)
```

Applying this through `Just (Just 1)`:

```
(fmap (1+)) (Just (Just 1))
= Just (fmap (1+) (Just 1))
= Just (Just (1+1))
```

The `fmap` doesn't fall through, and we reach an error when attempting to
evaluate `1 + (Just 1)`. More generally:

Let `f0` and `f1` be two `Functor`s, and `g` be a function with type signature
`g :: ((f a) -> (f b)) -> f (f a) -> f (f b)`. We can compose `f0` and `f1` to
make `(f0 . f1) :: f (f a)`. We know this is a container, but in order for it to
be a `Functor` we must be able to apply the `fmap` definition. Using this
composition and the function `g`, we have `fmap g (f0 . f1)`. First, note that
`g` and `f0 . f1` satisfy the requirements of `fmap`'s type signature. Then,
evaluating the result of this call to `fmap`, we see that the `Functor` wrapper
falls through and everything works (how to word this hmm, probably need a basic
inductive proof).


Laws
====

```hs
fmap id = id
fmap (g . h) = (fmap g) . (fmap h)
```

Exercises
---------

1. Although it is not possible for a `Functor` to satisfy the first `Functor`
   law but not the second (excluding `undefined`), the reverse is possible. Give
   an example of a (bogus) `Functor` instance which satisfies the second law but
   not the first.

--

```hs
data Bool' a = True' | False'

instance Functor Bool' where
    fmap _ _ = False'
```

This `fmap` sends all inputs to `False'`, which means that the second law will
always hold (since all function applications will result in `False'`) and the
first will not, since the identity function would send `True'` to `False'`.

---

2. Which laws are violated for the evil `Functor` instance shown below: both
   laws, or the first law alone? Give specific counterexamples.

```hs
-- Evil Functor instance
instance Functor [] where
  fmap _ [] = []
  fmap g (x:xs) = g x : g x : fmap g xs
```

--

Both laws are violated. For the first law, applying the identiy duplicates every
element of the list, which changes the structure of the container; for the
second law, applying the composition of two functions doubles the size of the
list, while applying the two functions individually quadruples the size of the
list. 

```hs
-- simple list implementation
data L a = E | C a (L a) deriving (Show)

-- 'evil' Functor instance
instance Functor L where
    fmap _ E = E
    fmap g (C a l) = C (g a) (C (g a) (fmap g l))
```

**Violates first law:**
```ghci
Prelude> fmap (0+) (C 2 (C 3 E))
C 2 (C 2 (C 3 (C 3 E)))
```

**Violates second law:**

```ghci
Prelude> fmap ((1+) . (2*)) (C 4 (C 3 E))
C 9 (C 9 (C 7 (C 7 E)))
Prelude> (fmap (1+) . fmap (2*)) $ (C 4 (C 3 E))
C 9 (C 9 (C 9 (C 9 (C 7 (C 7 (C 7 (C 7 E)))))))
```

