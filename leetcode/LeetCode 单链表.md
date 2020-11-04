# 单链表

## 206. 反转链表

反转一个单链表。

示例:

```
输入: 1->2->3->4->5->NULL
输出: 5->4->3->2->1->NULL
```
### 我的解法

```java
class Node(var value: Int = 0) {
    var next: Node? = null
}

fun reverse(head: Node?): Node? {
    if (head == null) return null

    var newHead: Node? = head
    var newMiddle: Node? = head
    var newTail: Node? = null
    while (newHead != null) {
        // 头节点进一
        newHead = newHead.next
        // 中节点指向后端
        newMiddle?.next = newTail

        // 重设尾节点
        newTail = newMiddle
        // 重设中节点
        newMiddle = newHead
    }

    // 注意在结束的时候，头和中都是null，返回尾节点就好
    return newTail
}
```
### 递归方式
```java
fun reverse(head: Node?): Node? {
    // 跳出递归的条件
    if (head == null) return null
    if (head.next == null) return head

    // 首链表之后整体进行反转
    val newHead = reverse(head.next)
    
    // 将首部进行反转
    head.next?.next = head
    // 这个是一开始写的，犯错在于没认清 头部在那里，总是把 newHead 作为 oldHead的next节点，但其实是最新的头部
    //newHead?.next = head
    // 置空 首部next
    head.next = null

    return newHead
}

```
### 思考
对于单链表，还是使用双指针更为清晰些。不过要想清楚链断裂和链接的顺序。

考虑使用递归方式，有奇效。

## 141. 环形链表
给定一个链表，判断链表中是否有环。

为了表示给定链表中的环，我们使用整数 pos 来表示链表尾连接到链表中的位置（索引从 0 开始）。 如果 pos 是 -1，则在该链表中没有环。


示例 1：
```
输入：head = [3,2,0,-4], pos = 1
输出：true
解释：链表中有一个环，其尾部连接到第二个节点。
```
这个题的示例的意思是 有了 head 和 pos，实际上测试的链表为 [3，2，0，-4，2，0，-4，2.....] 这样的。

现在给我们一个链表，我们怎么直到它是不是环的？

### 我的解法


## 142. 环形链表 II

给定一个链表，返回链表开始入环的第一个节点。 如果链表无环，则返回 null。

为了表示给定链表中的环，我们使用整数 pos 来表示链表尾连接到链表中的位置（索引从 0 开始）。 如果 pos 是 -1，则在该链表中没有环。

说明：不允许修改给定的链表。


示例 1：
```
输入：head = [3,2,0,-4], pos = 1
输出：tail connects to node index 1
解释：链表中有一个环，其尾部连接到第二个节点。
```

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