---
title: 在Linux内核中时常见到内嵌汇编语言的函数
layout: post
categories: Linux
tags: linux 内嵌汇编 asm
excerpt: 在Linux内核中，我们时常见到内嵌汇编语言的函数,那么内嵌汇编是怎么运行的
---

## 1. 内联汇编做了什么
在Linux内核中，我们时常见到内嵌汇编语言的函数，向这种在C语言函数内部嵌入汇编代码的做法我们称之为内联汇编。尽管现在的编译器对程序的优化都很优秀了，但在某些对效率要求较高的场合仍需要手动优化，我们暂时先不关心具体的内联汇编的规则，先来看看内联汇编是如何提高效率的。

## 1.1. 优化对比测试
测试环境：imx6ull
编译器：arm-linux-gnueabihf-gcc
反汇编：arm-linux-gnueabihf-objdump
Makefile:
```sh
test: main.c
        arm-linux-gnueabihf-gcc -o $@ $^

        arm-linux-gnueabihf-objdump -D test > test.dis
clean:
        rm test *.dis
```

## 1.1.1. 非内联汇编
写测试函数如下： main.c:
```sh
#include <stdio.h>
#include <stdlib.h>

int add(int a, int b)
{
	return a + b;
}

int main(int argc, char **argv)
{
	int a;
	int b;
	char *endptr;

	if (argc != 3)
	{
		printf("Usage: %s <val1> <val2>\n", argv[0]);
		return -1;
	}

	a = (int)strtol(argv[1], NULL, 0);
	b = (int)strtol(argv[2], NULL, 0);

	printf("%d + %d = %d\n", a, b, add(a, b));	
}

```


查看反汇编：
这是我们写的add函数：
```sh
00010404 <add>:
   10404:       b480            push    {r7}
   10406:       b083            sub     sp, #12
   10408:       af00            add     r7, sp, #0
   1040a:       6078            str     r0, [r7, #4]
   1040c:       6039            str     r1, [r7, #0]
   1040e:       687a            ldr     r2, [r7, #4]
   10410:       683b            ldr     r3, [r7, #0]
   10412:       4413            add     r3, r2
   10414:       4618            mov     r0, r3
   10416:       370c            adds    r7, #12
   10418:       46bd            mov     sp, r7
   1041a:       f85d 7b04       ldr.w   r7, [sp], #4
   1041e:       4770            bx      lr

```

然后在main函数中被调用：
```sh
00010420 <main>:
#...省略
10470:       f7ff ffc8       bl      10404 <add>
#...省略

```

## 1.1.2. 内联汇编
写测试函数如下： main.c:
```sh
#include <stdio.h>
#include <stdlib.h>

int add(int a, int b)
{
	int sum;
	__asm__ volatile (
		"add %0, %1, %2"
		:"=r"(sum)
		:"r"(a), "r"(b)
		:"cc"
	);
	return sum;
}

int main(int argc, char **argv)
{
	int a;
	int b;
	
	if (argc != 3)
	{
		printf("Usage: %s <val1> <val2>\n", argv[0]);
		return -1;
	}

	a = (int)strtol(argv[1], NULL, 0);	
	b = (int)strtol(argv[2], NULL, 0);	

	printf("%d + %d = %d\n", a, b, add(a, b));
	return 0;
}

```


```sh
00010474 <add>:
   10474:       1840            adds    r0, r0, r1
   10476:       4770            bx      lr

```

然后在main函数中被调用：
```sh
00010404 <main>:
#...省略
 10454:       f000 f80e       bl      10474 <add>
#...省略

```

## 1.2. 结论
可见，实现相同的功能，内联汇编add函数的指令相比纯C语言写的add函数大大简化，效率自然大大提高。

## 2. 语法规则
## 2.1. 内联汇编规则
```sh
asm [Qualifiers] (
                  ``AssemblerTemplate``
                  : ``OutputOperands``
                  : ``InputOperands``
                  : ``Clobbers`` 
                )

asm [Qualifiers] goto (
                      ``AssemblerTemplate``
                      : /* No outputs. */
                      : ``InputOperands``
                      : ``Clobbers``
                      : ``GotoLabels``
                    )

```

## 2.2. 示例
```sh
#include <stdio.h>
#include <stdlib.h>

int test(int a, int b)
{
	__asm__ goto (

		"cmp %1, %0\n\t"
    /* BLE指令：b比a小则跳转 */
		"blt %l2"
		: /* No outputs. */
		: "r"(a), "r"(b)
		: "cc"
		: carry);

	return 0;

carry:
	return 1;
}

int main(int argc, char **argv)
{
	int a;
	int b;

	if (argc != 3)
	{
		printf("Usage: %s <val1> <val2>\n", argv[0]);
		return -1;
	}

	a = (int)strtol(argv[1], NULL, 0);
	b = (int)strtol(argv[2], NULL, 0);

	printf("test return is %d\n",test(a, b));
	return 0;
}

```

## 2.3. imx6ull测试结果
BLE指令实现功能：b比a小则跳转
```sh
./test  1 2
test return is 0

./test  3 2
test return is 1

./test  2 2
test return is 0

```


