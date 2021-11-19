---
title: 基于CPS变换的尾递归转换算法
categories: 算法
tags:
  - js
---


# 前言

众所周知，**递归函数容易爆栈**，究其原因，便是函数调用前需要先将参数、运行状态压栈，而**递归则会导致函数的多次无返回调用，参数、状态积压在栈上，最终耗尽栈空间**。

一个解决的办法是从算法上解决，把递归算法改良成只依赖于少数状态的迭代算法，然而此事知易行难，线性递归还容易，树状递归就难以转化了，而且并不是所有递归算法都有非递归实现。

在这里，我介绍一种方法，**利用`CPS变换`，把任意递归函数改写成尾调用形式，以`continuation`链的形式，将递归占用的栈空间转移到堆上，避免爆栈的悲剧**。  
_需要注意的是，这种方法并不能降低算法的时间复杂度，若是指望此法缩短运行时间无异于白日做梦_

下文先引入**尾调用、尾递归、`CPS`等概念**，然后介绍**`Trampoline`技法**，将尾递归转化为循环形式（无尾调用优化语言的必需品），再**以`sum`、`Fibonacci`为例子讲解`CPS变换`过程**（虽然这两个例子可以轻易写成迭代算法，没必要搞这么复杂，但是最为常见好懂，因此拿来做例子，免得说题目都得说半天），最后讲**通用的`CPS变换`法则**

看完这篇文章，大家可以去看看`Essentials of Programming Languages`相关章节，可以有更深的认识

文中代码皆用`JavaScript`实现

# 尾调用 && 尾递归

先来探讨下在什么情况下函数调用才需要保存状态

像`Add(1, 2)`、`MUL(1, 2)`这种明显不需要保存状态，

像`Add(1, MUL(1, 2))`这种呢？计算完`MUL(1, 2)`后需要返回结果接着计算`Add`，因此计算`MUL`前需要保存状态

**由此，可以得到一个结论，只有函数调用处于参数位置上，调用后需要返回的函数调用才需要保存状态**，上面的例子中，`Add`是不需要保存状态，`MUL`需要保存

**尾调用指的就是，无需返回的函数调用，即函数调用不处于参数位置上**，上面的例子中，`Add`是尾调用，`MUL`则不是  
**写成尾调用形式有助于编译器对函数调用进行优化**，对于有尾调用优化的语言，只要编译器判断为尾调用，就不会保存状态

**尾递归则是指，写成尾调用形式的递归函数**，下面是一例


``` typescript
fact_iter = (x, r) => x == 1 ? 1 : fact_iter(x-1, x*r)
```

而下面的例子则不是尾递归，因为`fact_rec(x-1)`处于`*`的第二个参数位置上

``` typescript
fact_rec = x => x == 1 ? 1 : x * fact_rec(x-1)
```

因为尾递归无需返回，结果只跟传入参数有关，因此只需用少量变量记录其参数变化，便能轻易改写成循环形式，因此**尾递归和循环是等价**的，下面把fact_iter改写成循环：

``` typescript
function fact_loop(x)
{
    var r = 1

    while(x >= 1)
    {
        r *= x
        x--;
    }

    return r;
}
```

# CPS ( Continuation Passing Style )

要解释`CPS`，便先要解释`continuation`，  
**`continuation`是程序控制流的抽象，表示后面将要进行的计算步骤**

比如下面这段阶乘函数

``` typescript
fact_rec = x => x == 1 ? 1 : x * fact_rec(x-1)
```

显然，计算fact_rec(4)之前要先计算fact_rec(3)，计算fact_rec(3)之前要先计算fact_rec(2)，...  
于是，可以得到下面的计算链：


``` typescript
1 ---> fact_rec(1) ---> fact_rec(2) ---> fact_rec(3) ---> fact_rec(4) ---> print
```

展开计算链后，再从前往后执行，就可以得到最终结果。

对于链上的任意一个步骤，在其之前的是历史步骤，之后的是将要进行的计算，因此之后的都是`continuation`  
比如，对于`fact_rec(3)`，其`continuation`是`fact_rec(4) ---> print`  
对于`fact(1)`，其`continuation`是`fact_rec(2) ---> fact_rec(3) ---> fact_rec(4) ---> print`

当然，上面的计算链不需要我们手工展开和运行，**程序的控制流已经由语法规定好，我们只需要按语法写好程序，解释器自动会帮我们分解计算步骤并按部就班地计算**

然而，当现有语法无法满足我们的控制流需求怎么办？比如我们想从一个函数跳转至另一个函数的某处执行，**语言并没有提供这样的跳转机制，那便需要手工传递控制流**了。

**`CPS`是一种显式地把`continuation`作为对象传递的`coding`风格，以便能更自由地操控程序的控制流**

既然是一种风格，自然需要有约定，**`CPS`约定：每个函数都需要有一个参数`kont`，`kont`是`continuation`的简写，表示对计算结果的后续处理**

比如上面的`fact_rec(x)`就需要改写为`fact_rec(x, kont)`，读作 “计算出`x`阶乘后，用`kont`对阶乘结果做处理”

`kont`同样需要有约定，因为`continuation`是对某计算阶段结果做处理的，因此**规定`kont`为一个单参数输入，单参数输出的函数**，即`kont`的类型是`a->b`

因此，按`CPS`约定改写后的`fact_rec`如下：

``` typescript
fact_rec = (x, kont) => x == 1 ? kont(1) : fact_rec(x-1, res => kont(x*res))
```

当我们运行`fact_rec(4, r=>r)`，就可以得到结果`24`

模拟一下`fact_rec(3, r=>r)`的执行过程，就会发现，**解释器会先将计算链分解展开**：

``` typescript
fact_rec(3, r=>r)
fact_rec(2, res => (r=>r)(3*res))
fact_rec(1, res => (res => (r=>r)(3*res))(2*res))
(res => (res => (r=>r)(3*res))(2*res))(1)
```

当然，这种风格非常**反人类**，因为**内层函数被外层函数的参数分在两端包裹住，不符合人类的线性思维**

我们写成下面这种符合直觉的形式

``` typescript
1 ---> res => 2*res ---> res => 3*res ---> res => res
```

链上每一个步骤的输出作为下一步骤的输入

**当解释器展开成上面的计算链后，便开始从左往右的计算，直到运行完所有的计算步骤**

需要注意到的是，**因为`kont`承担了函数后续所有的计算流程，因此不需要返回，所以对`kont`的调用便是尾调用**  
**当我们把程序中所有的函数都按`CPS`约定改写以后，程序中所有的函数调用就都变成了尾调用了**，而这正是本文的目的  
这个改写的过程就称为**CPS变换**

需要警惕的是，**`CPS变换`并非没有状态保存这个过程，它只是把状态保存到continuation对象中**，然后一级一级地往下传，因此**空间复杂度并没有降低，只是不需要由函数栈帧来承受保存状态的负担而已**

**`CPS`约定简约，却可显式地控制程序的执行，程序里各种形式的控制流都可以用它来表达（比如协程、循环、选择等）**，  
所以很多函数式语言的实现都采用了`CPS`形式，将语句的执行分解成一个小步骤一次执行，  
当然，**也因为`CPS`形式过于简洁，表达起来过于繁琐，可以看成一种高级的汇编语言**

# Trampoline技法

经过`CPS变换`后，递归函数已经转化成一条长长的`continuation`链

尾调用函数层层嵌套，永不返回，然而**在缺乏尾调用优化的语言中，并不知晓函数不会返回，状态、参数压栈依旧会发生，因此需要手动强制弹出下一层调用的函数，禁止解释器的压栈行为，这就是所谓的`Trampoline`**

因为`continuation`只接受一个结果参数，然后调用另一个`continuation`处理结果，因此我们**需要显式地用变量`v`、`kont`分别表示上一次的结果**、下一个`continuation`，然后在一个循环里不断地计算`continuation`，直到处理完整条`continuation`链，然后返回结果


``` typescript
function trampoline(kont_v)  // kont_v = { kont: ..., v: ... }
{
    while(kont_v.kont)
        kont_v = kont_v.kont(kont_v.v);

    return kont_v.v;
}
```

**`kont_v.kont`是一个`bounce`，每次执行`kont_v.kont(kont_v.v)`时，都会根据上次结果计算出本次结果，然后弹出下一级`continuation`，然后保存在对象`{v: ..., kont: ...}`里**

**当然，在`bounce`中用`bind`的话，就不需要构造对象显式保存`v`了，因为`bind`会将`v`保存到闭包中，此时，`trampoline`变成：**


``` typescript
function trampoline(kont)
{
    while(typeof kont == "function")
        kont = kont();
    return kont.val;
}
```

用`bind`改写会更简洁，**然而，因为想要求的值有可能是个`function`，我们需要在`bounce`里用对象`{val: ...}`把结果包装起来**

具体应用可看下面的例子

# 线性递归的CPS变换：求和

**求和的递归实现：**


``` typescript
sum = x => { if(x == 0) return 0; else return x + sum(x-1) }
```

当参数过大，比如`sum(4000000)`，提示`Uncaught RangeError: Maximum call stack size exceeded`，爆栈了！

**现在，我们通过`CPS变换`，将上面的函数改写成尾递归形式：**

首先，**为`sum`多添加一个参数表示`continuation`，表示对计算结果进行的后续处理，**


``` typescript
sum = (x, kont) => ...
```

其中，`kont`是一个单参数函数，形如 `res => ...`，表示对结果`res`的后续处理

然后**逐情况考虑**：

当`x == 0`时，计算结果直接为`0`，并将`kont`应用到结果上,


``` typescript
sum = (x, kont) => { if(x == 0) return kont(0); else ... }
```

当`x != 0`时，需要先计算`x-1`的求和，然后将计算结果与`x`相加，然后把相加结果输入`kont`中，

``` typescript
sum = (x, kont) => { 
       if(x == 0) return kont(0); 
       else return sum( x - 1, res => kont(res + x) ) };
}
```

**好了，现在我们已经完成了`sum`的`CPS变换`，大家仔细看看，上面的函数已经是尾递归形式啦。**

现在还有最后的问题，**怎么去调用？**比如要算`4的求和`，`sum(4, kont)`，这里的`kont`应该是什么呢？

可以这样想，当我们计算出结果，后续的处理就是把结果简单地输出，因此**`kont`应为`res => res`**



``` typescript
sum(4, res => res)
```

把上面的代码复制到`Console`，运行就能得到结果`10`

**下面我们模拟一下sum(3, res => res)的运作，以对其有个直观的认识**

``` typescript
sum( 3, res => res )
sum( 2, res => ( (res => res)(res+3) ) )
sum( 1, res => ( res => ( (res => res)(res+3) ) )(res+2) ) )
sum( 0, res => ( res => ( res => ( (res => res)(res+3) ) )(res+2) ) )(res+1) )

// 展开continuation链
( res => ( res => ( res => ( (res => res)(res+3) ) )(res+2) ) )(res+1) )(0)

// 收缩continuation链
( res => ( res => ( (res => res)(res+3) ) )(res+2) )(0+1)
( res => ( (res => res)(res+3) ) )(0+1+2)
(res => res)(0+1+2+3)
6
```

从上面的展开过程可以看到，**`sum(x, kont)`分为两个步骤**：

* **展开`continuation`链**，尾调用函数层层嵌套，先做的`continuation`在外层，后做的`continuation`放内层，这也是**`CPS`反人类的原因**，**人类思考阅读都是线性**的（从上往下，从左往右），而**`CPS`则是从外到内，而且外层函数和参数包裹着内层**，阅读时还需要眼睛在左右两端不断游离

* **收缩`continuation`链**，不断将外层`continuation`计算的结果往内层传

当然，现在运行`sum(4000000， res => res)`，依然会爆栈，因为`js`默认并没有对尾调用做优化，**我们需要利用上面的`Trampoline`技法将其改成循环形式**（上文已经提过，尾递归和循环等价）

可是等等，上面说的`Trampoline`技法只针对于收缩`continuation`链过程，可是`sum(x, kont)`还包括展开过程啊？别担心，**可以看到展开过程也是尾递归形式，我们只需稍作修改，就可以将其改成`continuation`的形式**：

``` typescript
( r => sum( x - 1, res => kont(res + x) )(null)
```

**如此便可把`continuation`链的展开和收缩过程统一起来，写成以下的循环形式**：

``` typescript
function trampoline(kont_v)
{
    while(kont_v.kont)
        kont_v = kont_v.kont(kont_v.v);

    return kont_v.v;
}

function sum_bounce(x, kont)
{    
    if(x == 0) return {kont: kont, v: 0};
    else return { kont: r => sum_bounce(x - 1, res => {
                                                 return { kont: kont, 
                                                          v: res + x }
                                               } ),
                  v: null };
}

var sum = x => trampoline( sum_bounce(x, res => 
                                            {return { kont: null, 
                                                      v: res } }) )
```

OK，**以上便是改成循环形式的尾递归写法**，  
把`sum(4000000)`输入`Console`，稍等片刻，便能得到答案`8000002000000`

**当然，用`bind`的话可以改写成更简约的形式：**



``` typescript
function trampoline(kont)
{
    while(typeof kont == "function")
        kont = kont();
    return kont.val;
}

function sum_bounce(x, kont)
{    
    if(x == 0) return kont.bind(null, {val: 0});
    else return sum_bounce.bind( null, x - 1, res => kont.bind(null, {val: res.val + x}) );
}

var sum = x => trampoline( sum_bounce(x, res => res) )
```

也能起到同样的效果

# 树状递归的CPS变换：Fibonacci

因为`Fibonacci`是`树状递归`，转换起来要比线性递归的`sum`麻烦一些，先**写出普通的递归算法**：


``` typescript
fib = x => x == 0 ? 1 : ( x == 1 ? 1 : fib(x-1) + fib(x-2) )
```

同样，当参数过大，比如`fib(40000)`，就会爆栈

**开始做`CPS变换`**，有前面例子铺垫，下面只讲关键点

**添加`kont`参数**，则`fib = (x, kont) => ...`

**分情况考虑**

**当`x == 0 or 1`**，`fib = (x, kont) => x == 0 ? kont(1) : ( x == 1 ? kont(1) ...`

**当`x != 1 or 1`**，需要先计算`x-1`的`fib`，再计算出`x-2`的`fib`，然后将两个结果相加，然后将kont应用到相加结果上



``` typescript
fib = (x, kont) => 
      x == 0 ? kont(1) : 
      x == 1 ? kont(1) : 
               fib( x - 1, res1 => fib(x - 2, res2 => kont(res1 + res2) ) )
```

**以上便是`fib`经`CPS变换`后的尾递归形式**，可见**难点在于`kont`的转化**，这里需要好好揣摩

**最后利用`Trampoline`技法将尾递归转换成循环形式**

``` typescript
function trampoline(kont_v) {
  while (kont_v.kont)
    kont_v = kont_v.kont(kont_v.v);

  return kont_v.v;
}

function fib_bounce(x, kont) {
  if (x == 0 || x == 1) return { kont: kont, v: 1 };
  else return {
    kont: r => fib_bounce(x - 1,
      res1 => {
        return {
          kont: r => fib_bounce(x - 2,
            res2 => {
              return {
                kont: kont,
                v: res1 + res2
              };
            }),
          v: null
        };
      }),
    v: null
  };
}

var fib = x => trampoline(fib_bounce(x, res => {
  return {
    kont: null,
    v: res
  };
}));
```

OK，**以上便是改成循环形式的尾递归写法**，  
在`console`中输入`fib(5)`、`fib(6)`、`fib(7)`可以验证其正确性，

当然，**当你运行`fib(40000)`时，发现的确没有提示爆栈了，但是程序却卡死了，何也？**

正如我在前言说过，这种方法并不会降低树状递归算法的时间复杂度，只是将占用的栈空间以闭包链的形式转移至堆上，免去爆栈的可能，但是**当参数过大时，运行复杂度过高，`continuation`链过长也导致大量内存被占用**，因此，优化算法才是王道

**当然，用`bind`的话可以改写成更简约的形式：**

``` typescript
function trampoline(kont){
    while(typeof kont == "function")
        kont = kont();
    return kont.val;
}

fib_bounce = (x, kont) =>
 x == 0 ? kont.bind(null, {val: 1}) : 
 x == 1 ? kont.bind(null, {val: 1}) : 
          fib_bounce.bind( null, x - 1, 
                           res1 => fib_bounce.bind(null, x - 2,
                                                   res2 => kont.bind(null, {val: res1.val + res2.val}) ) )

var fib = x => trampoline( fib_bounce(x, res => res) )
```

也能起到同样的效果

# CPS变换法则

对于基本表达式如数字、变量、函数对象、参数是基本表达式的内建函数(如四则运算等)等，不需要进行变换，

若是函数定义，则需要添加一个参数`kont`，然后对函数体做`CPS变换`

若是参数位置有函数调用的函数调用，`fn(simpleExp1, exp2, ..., expn)`，如`exp2`就是第一个是函数调用的参数  
则过程比较复杂，用伪代码表述如下：(`<<...>>`内表示表达式， `<<...@exp...>`表示对exp求值后再代回`<<...>>`中)：


``` typescript
cpsOfExp(<< fn(simpleExp1, exp2, ..., expn) >>, kont)
= cpsOfExp(exp2, << r2 => @cpsOfExp(<< fn(simpleExp1, r2, ..., expn) >>, kont) >>)
```

顺序表达式的变换亦与上类似

当然这个问题不是这么容易讲清楚，首先你需要对你想要变换的语言了如指掌，知道其表达式类型、求值策略等，  
`JavaScript`语法较为繁杂，解释起来不太方便，  
之前我用`C++`模板写过一个`CPS`风格的`Lisp`解释器，日后有时间以此为例详细讲讲