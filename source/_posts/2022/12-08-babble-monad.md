"title": "乱谈 Monad",
"date": "2022/12/08 20:46:25",
"categories": ["lambda"]
;;;

## Preface

这几天按着我半吊子的 haskell 水平稍微研究了一下 monad，还是有点不是很理解，只能浅浅地谈一谈我自己的想法，错误和纰漏难以避免，所谓乱谈是也。

## The Moand

这部分尽量会详细地阐述一下[刘月半大神的这篇文章](https://www.zhihu.com/question/19635359/answer/172074046)，适当的改动了一些例子，首先从 `Maybe` 的构造函数开始，

```haskell
data Maybe a = Just a | Nothing
```

其中 `a` 是一个类型，我们可以“实例化”一个 `Maybe Int` 类型出来等等，并且存在两个“构造器”，`Just` 和 `Nothing`。例如我们可以构造一个 `safeDiv` 函数，这个函数作用是保证在遇到除零错误时不会让程序 crash，而是返回一个 `Maybe Int` 类型：

```haskell
safeDiv :: Int -> Int -> Maybe Int
safeDiv _ 0 = Nothing
safeDiv x y = Just (x `div` y)	-- 除法运算符 / 不能作用在 Int 上
```

如果我们希望一个比较复杂的除法运算中，都能保证除零安全的话，我们可能会写出下面这样的代码：

```haskell
-- 安全地计算 a/b/c/d/e
case safeDiv a b of
  Nothing -> Nothing
  Just x  -> case safeDiv x c of	-- 如果 b != 0，能够继续进行除法，尝试计算 (a/b)/c
    Nothing -> Nothing
    Just y  -> case safeDiv y d of	-- 如果 c != 0，继续尝试计算 (a/b/c)/d
      Nothing -> Nothing
      Just z  -> safeDiv z e		-- 如果 d != 0，继续尝试计算 (a/b/c/d)/e
```

可以看到代码非常的啰嗦，而且看着很难受（对应过程式语言就是一层又一层的 if 嵌套），有没有办法能优化呢？实际上上面的例子中，我们可以从几个 `case` 中提取出一些公共特征（伪代码，`{Int}` 表示此处的变量类型为 `Int`）：

```haskell
-- 将 safeDiv x c 使用 \x -> safeDiv x c 替换，即可得到最简洁的 Int -> Maybe Int 函数；
--   因为被 lambda 捕获的量基本都能被视作常量（？）
case {Maybe Int} of
  Nothing -> Nothing
  Just x -> {Int -> Maybe Int} x
```

给上面的类型附上名字和类型，就成了：

```haskell
fmap' :: (Int -> Maybe Int) -> Maybe Int -> Maybe Int
fmap' f m = case m of
  Nothing -> Nothing
  Just x -> f x
```

到这里我们就得到了一个 `Maybe` 的所谓 functor（函子），

> Maybe 之所以是函子，是因为它可以通过 fmap（functor map）把所有的函数拎到「Maybe 空间」里。

有了函子之后，我们尝试把这个抽象应用到我们的例子上去看看：

```haskell
let
    x = safeDiv a b			-- 第一层 a/b
    y = fmap' (\v -> safeDiv v c) x	-- 第二层 (a/b)/c
    z = fmap' (\v -> safeDiv v d) y	-- 第三层 (a/b/c)/d
in
    fmap' (\v -> safeDiv v e) z		-- 第四层 (a/b/c/d)/e
```

抛开性能不谈（雾），这种写法看起来已经舒服很多了，但是感觉还是“存在进步空间”的样子。也的确，haskell 给我们提供了一个 `>>=` 运算符，可以让我们来实现 `fmap`，并且再次优化上面的代码：

```haskell
-- 这是一个中缀运算符，这个例子中 (>>=) m f 等价于 m >>= f；定义与 fmap 完全一致
(>>=) :: Maybe Int -> (Int -> Maybe Int) -> Maybe Int
(>>=) m f = case m of
  Nothing -> Nothing
  Just x -> f x

-- the optimzed version
safeDiv a b >>= (\v ->
  safeDiv v c >>= (\w ->
    safeDiv w d >>= (\x ->
      safeDiv x e)))
```

好像，有点反向优化了？没事，haskell 为这个模式设计了 `do` 语法糖：

```haskell
val = do
  x <- safeDiv a b
  y <- safeDiv x c
  z <- safeDiv y d
  safeDiv z e
```

简洁，好看，elegant！当然，因为标准库里[已经提供](https://hackage.haskell.org/package/base-4.17.0.0/docs/Data-Maybe.html)了 `instance Monad Maybe` 了，所以最后的代码中，我们不需要再定义任何其他函数了，只需要保留 `safeDiv` 就可以了。当然我们也能自行实现各种各样的 functor，有点像是 OOP 的接口一样（笑）。

## Functor and Monad

上面的例子中，我们的 `fmap'` 是自己定义的，而官方针对 [`Maybe` 实现的 `fmap`](https://hackage.haskell.org/package/base-4.17.0.0/docs/src/GHC.Base.html) 是不太一样的：

```haskell
-- | @since 2.01
instance  Functor Maybe  where
    fmap _ Nothing       = Nothing
    fmap f (Just a)      = Just (f a)	-- notice here!
```

我们的例子中模式匹配项是 `Just x -> fx`，但是官方的却是 `Just x -> Just (f a)`，后者其实才是标准格式；`safeDiv` 这个例子不同是因为我们完全是根据函数类型来“特化”的一个：

```haskell
-- safeDiv version of fmap
fmap :: (Int -> Maybe Int) -> Maybe Int -> Maybe Int

-- haskell fmap
fmap :: (a -> b) -> f a -> f b
```

Obviously，区别就出来了。这里提这个区别是因为这里要开始区分 functor 和 monad 了。前文的例子中可能大家会觉得 monad 跟 functor 是一回事，实际上是存在差异的。Haskell 定义了 [Functor-Applicative-Monad](https://wiki.haskell.org/Functor-Applicative-Monad_Proposal) 结构，简而言之就是 applicative 必定是 functor，monad 必定是 applicative，三者有继承关系，所以 functor 其实是性质是比较弱的。

这部分主要会根据哔站[鹤翔万里的视频](https://www.bilibili.com/video/BV1qh411q7cS)进行一定的细致阐述，也会对前文刘月半大神的文章进行再补充。先从 functor 开始吧，它的作用看上面函数定义就能略知一二了，给定一个一般的函数，让它去处理“装箱”后的类型，并且返回一个同样被装箱了的类型，看 haskell 的 `Maybe` 实现就知道了，这里贴出来视频的几个例子：

```haskell
-- instance Functor Maybe
fmap (+3) (Just 1) = Just 4
fmap (+3) Nothing = Nothing

-- instance Functor []
fmap (+3) [1, 2, 3] = [4, 5, 6]
```

当然 functor 根据定义，它的局限性就在于只能处理值，对于函数，它只会返回一个装箱后的函数：

```haskell
fmap (+) (Just 3) = Just (+3)
```

并且我们无法继续往下进行了：

```haskell
fmap (Just (+3)) (Just 3) = 🤯
```

这时就需要引入 applicative 来处理了，直接看官方定义：

```haskell
-- | @since 2.01
instance Applicative Maybe where
    pure = Just

    Just f  <*> m       = fmap f m
    Nothing <*> _m      = Nothing

    liftA2 f (Just x) (Just y) = Just (f x y)
    liftA2 _ _ _ = Nothing

    Just _m1 *> m2      = m2
    Nothing  *> _m2     = Nothing
```

主要关注 `<*>` 这个函数（读作 apply），可以看到 applicative 将 `Just (+3)` 里面的 `(+3)` 函数给取出来了，然后通过 `fmap` 应用函数，这样得到的结果依然是个 `Maybe` 类型，it just works。

同样，贴上视频中的几个例子当作 memo：

```haskell
-- instance Applicative Maybe
Just (+3) <*> Just 1 = Just 4

-- instance Applicative []
[(+1), (*2)] <*> [2, 3] = [3, 4, 4, 6]
```

如果我们把前文例子中 `safeDiv` 里的 `fmap'` 删去，那么下面这个函数就会开始出类型错误：

```haskell
-- without customized fmap function
let
    x = safeDiv a b			-- 第一层 a/b
    y = fmap (\v -> safeDiv v c) x	-- 第二层 (a/b)/c
    z = fmap (\v -> safeDiv v d) y	-- 第三层 (a/b/c)/d
in
    fmap (\v -> safeDiv v e) z		-- 第四层 (a/b/c/d)/e
```

原因前文说了，`fmap` 其实返回的是 `Just (f a)`，也就是：

```haskell
fmap (\v -> safeDiv v 1) (Just 10) = Just (Just 10)
```

这就有点难受了，不是我们预期想要的 `Just 10`。这种情况下就需要使用 monad 了，也就是上面定义中 `>>=`（读作 bind）来进行处理了，也是直接看官方代码吧：

```haskell
-- | @since 2.01
instance  Monad Maybe  where
    (Just x) >>= k      = k x
    Nothing  >>= _      = Nothing

    (>>) = (*>)
```

对比官方 `fmap` 就能看到，monad 的 instance 中不再是 `Just (k x)`，而是简介、明了的 `k x`，neat。这也是跟我们在 `safeDiv` 中实现的 `>>=` 版本完全等价，所以它就是一个 monad。

依然，贴上一些视频中的例子：

```haskell
Just 1 >>= \x -> Just (x+3) = Just 4
```

个人理解，无论是 functor、applicative 还是 monad，它们本质上都是为了更好地处理“被装箱的类型”。而 haskell 比较推崇这种盒子模式，很多类型像 `Maybe a`、`IO a` 这种，都实现了 monad 的 bind 运算符。

比如 Haskell 里常用的数组展开，其实也是借助 functor 来完成的，来自[评论区的 Narc 大神](https://zhuanlan.zhihu.com/p/39734882)：

```haskell
-- original
[ x + y | x <- [1..3], y <- [1..x]]

-- desuger
[1..3] >>= \x -> [1..x] >>= \y -> return (x + y)
```

另外 JavaScript 的 `Promise` 也是跟 monad 有点“异曲同工”之妙的，毕竟 lazy eval 能看作是异步（。

## Myth

我个人依然有一些迷思，可能需要系统地学习下范畴论才能解惑了。

1. 都说 monad 能隔离副作用，但是我看下，感觉跟闭包差不多？将所有状态装箱到一个闭包中，当然外部无法修改，[Stack Overflow 上也有讨论](https://stackoverflow.com/a/79077)，说区别在于“A monad is (roughly) more like a context in which functions can be chained together sequentially, and controls how data is passed from one function to the next.”
2. 对用法依然不是很清晰，暂时的想法是，monad 就是用来处理“盒子”的，side effect 可以装进盒子中，error handling 也可以装进去，等等；并且 monad 可以“传递”（不同于 closure 只能捕获函数内的，monad 可以捕获整个链式调用上的结果)，加上 `return` 函数的辅助，容易实现 chain call，并且 chain 上的函数都共享整个 context？
3. 一个自函子范畴上的幺半群。。嗯，还是不太懂😂

还有，原知乎帖子里讲述了一个 `join` 操作：

```haskell
let x = safeDiv a b
    y = fmap (safeDiv c) x
    z = (fmap . fmap) (safeDiv d) y
    in (fmap . fmap . fmap) (safeDiv e) z

join :: Maybe (Maybe a) -> Maybe a
  join (Just x) = x
  join Nothing  = Nothing

(>>=) :: Maybe a -> (a -> Maybe b) -> Maybe b
  (>>=) mbx f = join (fmap f mbx)
```

这一部分我暂时还不是很懂，答主给的例子中那一串 `fmap` 我暂时还是没能推导出类型出来（不太熟悉 `.` 操作符，是 haskell 超高校级的新手了）。To be continued。

其实自己还是有挺多地方不懂的，不过 haskell 可能暂时不会再学习了，目前对 erlang 更有兴趣一些，学一门语言等于学完了并发范式，不香吗不香吗（其实主要是数学太渣了，看了一小会范畴论就已经开始歇菜了🤧）。
