---
    layout: post
    title: 从emacs到lisp
---

## emacs
- [fork的emacs配置](https://github.com/lisp2c/emacs.d)
- 官方指南：C-h t 吧,[中文快速指南](../file/TUTORIAL.cn)
- 在emacs slime中: 换行键 C-j 

## lisp
- lisp可以认为是函数抽象和语义抽象的。lisp的语法很像AST(抽象语法树)
- 在《黑客与画家》那本书上就被安利了。[commonlisp的一本指导书](https://acl.readthedocs.io/en/latest/zhCN/ch1-cn.html)

- 理解递归 (Understanding Recursion)
    * 可以用数学归纳法来验证一个递归程序是否正确，而不是trace程序执行过程
        * 数学归纳法并不是只能应用于形如“对任意的n”这样的命题。对于形如“对任意的n=0,1,2,...,m”这样的命题，如果对一般的n比较复杂，而n=m比较容易验证，并且我们可以实现从k到k-1的递推，k=1,...,m的话，我们就能应用归纳法得到对于任意的n=0,1,2,...,m，原命题均成立。


