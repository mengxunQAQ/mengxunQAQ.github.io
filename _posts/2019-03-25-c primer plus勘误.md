---
layout:     post
title:      c primer plus勘误
subtitle:   程序17.2的一处错误
date:       2019-03-25
author:     mengxun
header-img: img/pipenv.jpg
catalog: true
tags:
    - C/C++
---

#### 逻辑错误

在第17章的第二个程序（17.2）里有一处释放堆的程序如下：

```c
/* 完成任务，释放已分配的内存 */
    current = head;
    while (current != NULL)
    {
        current = head;
        head = current -> next;
        free(current);
    }
```

上面的代码实现的功能是在 `malloc` 申请堆来存放链表之后，用 `free` 一个个释放空间。

但是这个代码里有一个小错误会导致运行出错，就是当释放完最后一个链表元素的时候，此时 `current` 指针并不是 `NULL`，所以程序仍会循环一次，导致最后 `current -> next` 相当于 `NULL -> next`，这样便会内存访问出错导致段错误。

#### 修改

可以通过把 `while (current != NULL) ` 修改成 `while (head != NULL)`来避免这个错误。

