---
layout: post
title: "[IOS] Switch scope"
date: 2014-08-06 23:48:35 +0900
comments: true
categories: IOS Objective-C
---

Objective c的switch中的错误信息：
```
Expected expression
```


在Objective c中，如果需要在switch语句中声明变量，需要将相关代码放在{}，否编译会报错。  
原因是在ARC（Automatic Reference Counting）时候，如果没有加{}，编译器不能确定这个变量的作用域。

所以需要加入{}，声明变量作用域。

下面，case1是会报错，case2可以通过。
```
switch(condition) {
    case 1:
        // error
        int a = 1;
        NSLog(@"a: %d", a);
        break;
    case 2:
    {
        int b = 1;
        NSLog(@"b: %d", b);
        break;
    }
    default:
        break;
}
```

## Reference
[stackoverflow:When converting a project to use ARC what does “switch case is in protected scope” mean?](http://stackoverflow.com/questions/7562199/when-converting-a-project-to-use-arc-what-does-switch-case-is-in-protected-scop)
