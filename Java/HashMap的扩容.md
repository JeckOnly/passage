```java
 for (int j = 0; j < oldCap; ++j) {// 遍历旧数组的每一个桶
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;// 释放旧数组引用
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);// 红黑树的不管
                    else { // 维护原有顺序
                        // 扩容数组大小*2
                        // 用两个指针来维护链表元素顺序
                        Node<K,V> loHead = null, loTail = null;// 留在原索引
                        Node<K,V> hiHead = null, hiTail = null;// 在原索引+oldCap的位置
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
```



可以看到，1.8之后，扩容时，保持原链表元素顺序不变，尾插法。（注：put方法(不分1.7、1.8)也是尾插法（答错过一次））

> 1.7扩容时是头插法，顺序会颠倒。