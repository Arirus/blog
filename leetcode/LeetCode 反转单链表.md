# 链表相关

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


