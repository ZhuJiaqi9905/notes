版本：redis-5.0.0

adlist.c, adlist.h: 双向链表

- 不带虚拟的头结点

```
void listEmpty(list *list);移除所有的元素，但是不释放链表本身。
```

