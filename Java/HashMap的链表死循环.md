![image-20220902110539153](../img/image-20220902110539153.png)



链表死循环后，put和get都有问题。

比如get（11）和put（11），get的话一直没有匹配却去不到null，put的话尾插也去不到null。