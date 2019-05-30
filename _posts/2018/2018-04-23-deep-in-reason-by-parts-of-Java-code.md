---
title: 由一段Java代码渗析原因
date: 2018-04-23 23:50:36
tags: ['Java']
categories: ['Java']
---

前段时间，看到了一段如下代码：
```Java
package com.jangz.syntax.scary;

public class ScaryExpression {

	public static void autoIncrement() {
		int count = 0;
		for (int i = 0; i < 100; i++)
			count = count++;
		System.out.printf("count is %d.\n", count);
	}

	public static void main(String[] args) {
		autoIncrement();
	}
}
```
然后这段代码的运行结果如下：
![2018-04-23_232405.png](https://upload-images.jianshu.io/upload_images/3631711-6b5a33cd9cb3e2bb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
<!-- more -->
很神奇的0，对不对？（而不是奇怪）毕竟作为Java程序员从入门开始就已经刻在脑子里面关于` i++ `与 ` ++i `的区别。

我们都知道前者是先赋值后自增，后者则是先自增后赋值。那底层到底是怎么样的呢？好，下面咱们就通过反编译代码，看看底层字节码指令是什么样的。

我们先执行如下操作，得到data.txt反编译代码：
![2018-04-23_232047.png](https://upload-images.jianshu.io/upload_images/3631711-9ddfc3c2548974e9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

现在我们看看data.txt里面都是些什么东西：
```
Compiled from "ScaryExpression.java"
public class com.jangz.syntax.scary.ScaryExpression {
  public com.jangz.syntax.scary.ScaryExpression();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void autoIncrement();
    Code:
       0: iconst_0
       1: istore_0
       2: iconst_0
       3: istore_1
       4: iload_1
       5: bipush        100
       7: if_icmpge     21
      10: iload_0
      11: iinc          0, 1
      14: istore_0
      15: iinc          1, 1
      18: goto          4
      21: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
      24: ldc           #3                  // String count is %d.\n
      26: iconst_1
      27: anewarray     #4                  // class java/lang/Object
      30: dup
      31: iconst_0
      32: iload_0
      33: invokestatic  #5                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
      36: aastore
      37: invokevirtual #6                  // Method java/io/PrintStream.printf:(Ljava/lang/String;[Ljava/lang/Object;)Ljava/io/PrintStream;
      40: pop
      41: return

  public static void main(java.lang.String[]);
    Code:
       0: invokestatic  #7                  // Method autoIncrement:()V
       3: return
}

```

一下子上来，看不懂？！有木有，没关系，咱们简化一下代码，通过下面一段代码来进行分析：
```Java
package com.jangz.syntax.scary;

public class ScaryExpressionExample {
	public static void main(String[] args) {
		int i = 1;
		i = i++;
		int j = 1;
		j = ++j;
	}
}
```
反编译代码如下：
```
Compiled from "ScaryExpressionExample.java"
public class com.jangz.syntax.scary.ScaryExpressionExample {
  public com.jangz.syntax.scary.ScaryExpressionExample();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: iconst_1
       1: istore_1
       2: iload_1
       3: iinc          1, 1
       6: istore_1
       7: iconst_1
       8: istore_2
       9: iinc          2, 1
      12: iload_2
      13: istore_2
      14: return
}
```
这次有没有简洁明了一些了，对吧？！好了，开始分析：
Code:
   0:   iconst_1  // 把1压入栈顶
   1:   istore_1  // 把栈顶的值放到局部变量1（即i）中
   2:   iload_1   // 把i的值放到栈顶，也就是说此时栈顶的值是1
   3:   iinc    1, 1  //把局部变量1（即i）加1变为2，注意这时栈中仍然是1，没有改变
   6:   istore_1    //把栈顶的值放到局部变量1中，即i这时候由2变成了1

到此处，以上便完成了``` int i = 1; i = i++; ```这两行代码。

   7:   iconst_1   // 把1压入栈顶
   8:   istore_2   // 把栈顶的值放到局部变量1（即i）中
   9:   iinc    2, 1 // 把局部变量2（即j）加1变为2，注意这时栈中仍然是1，没有改变
   12:  iload_2    // 把局部变量2（即j）的值放到栈顶，此时栈顶的值变为2
   13:  istore_2   // 把栈顶的值放到局部变量2中，即j这时候真正由1变成了2

到此时，便执行完毕` int j = 1; j = ++j; `这两行代码。

   14:  return

到这里，应该一目了然，明白为什么 执行` i = i++; `过后，i的值还是原先的值的原因了吧？！
那么咱么再回到最开始的那段for循环中count变量进行 `count++`并重新赋值给count的代码：
` count = count++；`这句话，Java虚拟机执行时时这样的：count的值加了1，但是栈中的值还是0，马上栈中的值覆盖掉了count中的值，即count变为0。因此，不管循环多少次，count都是0。但是，如果修改成` count = ++count;`，那么，结果就是100了。

好了，其实写这篇文章的目的就是为了说明一点，学习技术的时候，要学会理解记忆，而不要总是死记硬背，不会活学活用，不然，浪费的只是时间，到头来还仍然一知半解，不知所以然！望轻喷，不是不喷，哈哈......
