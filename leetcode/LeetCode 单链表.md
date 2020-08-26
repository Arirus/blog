# 单链表

### 我的解答

```java
fun hasCycle(head: ListNode?): Boolean {
    var result = false;
    if (head?.next == null) return false

    var pre = head
    var cur = head

    //两个指针没有后续指针，也就是有限长度的链表。这种情况肯定不是环链
    while (cur?.next != null && pre?.next?.next != null) {
        // 两个指针相遇，这时候肯定是环链
        if (pre.next?.next == cur.next) {
            result = true
            break
        }
        cur = cur.next

        pre = pre.next?.next
    }

    return result
}
```

判断是否有环，用快慢指针，当有环时，两个指针总会进入，因为步长差值为1，所以早晚会相遇的。


```java
fun detectCycle(head: ListNode?): ListNode? {
    if (head?.next == null) return null

    var pre = head
    var cur = head
    var indicator = head

    var hasCycle = false
    while (cur?.next != null && pre?.next?.next != null) {
        if (pre.next?.next == cur.next) {
            cur = cur.next
            hasCycle = true
            break
        }
        cur = cur.next

        pre = pre.next?.next
    }

    //相遇后，从头开始遍历一个，重合点就是起始点。
    if (hasCycle) {
        while (indicator != cur) {
            indicator = indicator?.next
            cur = cur?.next
        }
        return indicator
    }
    

    return null
}
```

而在相遇的时候，其步长差值肯定为一个环的长度。同时，慢指针肯定一圈还没走完。还差一部分，这差的一部分就是非环的一部分。从头部开始一个新指针，二者相遇之后就是环开始的地方。

时间复杂度O(n),空间复杂度O(1)

## 21. 合并两个有序链表
将两个升序链表合并为一个新的 升序 链表并返回。新链表是通过拼接给定的两个链表的所有节点组成的。 

示例：
```
输入：1->2->4, 1->3->4
输出：1->1->2->3->4->4
```

### 我的思路

```java
fun mergeTwoLists(l1: ListNode?, l2: ListNode?): ListNode? {
    var head: ListNode? = null
    var tail: ListNode? = null

    var head1 = l1;
    var head2 = l2;

    while (head1 != null || head2 != null) {
        if (head1 == null) {
            if (tail!=null) tail.next = head2
            tail = head2;
            if (head == null) head = tail
            break
        } else if (head2 == null) {
            if (tail!=null)
                tail.next = head1
            tail = head1
            if (head == null) head = tail
            break
        } else {
            if (head1.value < head2.value) {
                if (tail != null)
                    tail.next = head1

                tail = head1
                head1 = head1.next
            } else {
                if (tail != null)
                    tail.next = head2

                tail = head2
                head2 = head2.next
            }
        }
        if (head == null) head = tail

    }
    return head
}
```
主要是分三种情况，有1没2，有2没1，12都有。因为这种情况，是要判断新头结点的是否为空，因此这里可以做优化，减少判断条件。

```java
fun mergeTwoLists(l1: ListNode?, l2: ListNode?): ListNode? {

    var head: ListNode? = ListNode(-1)
    var tail: ListNode? = head

    var head1 = l1;
    var head2 = l2;

    while (head1 != null || head2 != null) {
        if (head1 == null) {
            tail?.next = head2;
            break
        } else if (head2 == null) {
            tail?.next = head1
            break
        } else {
            if (head1.`val` < head2.`val`) {
                tail?.next = head1
                head1 = head1.next
            } else {
                tail?.next = head2
                head2 = head2.next
            }
            tail = tail?.next
        }

    }

    return head?.next 
}
```

主要优化点就是新增了一个头结点，所有结点都链到这个后面。这样就不用增加额外的判断。

### 递归方式
```java
fun mergeTwoLists(l1: ListNode?, l2: ListNode?): ListNode? {

    if (l1 == null && l2 == null) return null
    if (l1 == null) return l2
    if (l2 == null) return l1


    val head =
            if (l1.`val` < l2.`val`)
                l1
            else
                l2


    head.next =
            if (head == l1)
                mergeTwoLists(l1.next, l2)
            else
                mergeTwoLists(l1, l2.next)


    return head
}
```

使用递归的话，每次只需要确定一个头结点，再将后面的链表进行递归处理就好。注意写好跳出条件。

## 24. 两两交换链表中的节点

给定一个链表，两两交换其中相邻的节点，并返回交换后的链表。

你不能只是单纯的改变节点内部的值，而是需要实际的进行节点交换。

 
示例:
```
给定 1->2->3->4, 你应该返回 2->1->4->3.
```

### 我的解法
```java
fun swapPairs(head: ListNode?): ListNode? {
    //当链表的结点不满足两个的时候，返回链表本身
    if (head?.next == null) return head

    //获取当前的链表的前两个结点
    var pre = head
    var cur = head.next

    //交换两个结点
    pre.next = cur?.next
    cur?.next = pre

    //后结点的next 指向新的处理过的链表
    pre.next = swapPairs(pre.next)

    // 返回
    return cur
}
```
模仿上面，毕竟迭代的总能转换成递归的。

```java
//迭代的方式
fun swapPairs(head: ListNode?): ListNode? {
    if (head?.next == null) return head
    var newHead = head.next

    var pre = head
    var cur = head.next
    var fake:ListNode? = ListNode(-1).apply {
        this.next = pre
    }

    while (cur != null && pre != null) {

        pre.next = cur.next
        cur.next = pre
        fake?.next = cur

        var tmp = pre
        pre = cur
        cur = tmp

        pre = cur.next
        cur = cur.next?.next
        fake = fake?.next?.next

    }

    return newHead

}
```
这里可以考虑进行优化：1.每次循环的时候，才进行相应的赋值；2.将 head 结点作为游标进行操作

```java
fun swapPairs(head: ListNode?): ListNode? {
    if (head?.next == null) return head
    var newHead = head.next

    var indicator = head

    var fake: ListNode? = ListNode(-1).apply {
        this.next = head
    }

    while (indicator?.next != null) {

        var cur = indicator
        var forward = indicator.next
        
        fake?.next = forward
        cur.next = forward?.next
        forward?.next = cur

        fake = cur
        indicator = cur.next

    }
    return newHead
}
```
这样更为美观