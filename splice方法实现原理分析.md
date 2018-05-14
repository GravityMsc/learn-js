## 函数原型：Array.prototype.splice(start, deleteCount, item1, item2, ...)

splice至少有1个参数，第一个参数start（必须）是开始插入或删除处的元素索引，deleteCount是要删除的元素个数，从start开始，包含start处的元素。如果deleteCount为0，那表示就是插入元素，将后面参数值插入到start位置之后，比如：[1, 2, 3].splice(1, 2)会删除原数组的arr[1], arr[2]处的元素，并将删除的数组装成一个数组返回。

那么JavaScript究竟是如何做到这一切的呢？通过翻阅[ECMA5.1文档](http://www.ecma-international.org/ecma-262/5.1/#sec-15.4.1)，我找到了答案。以下 是splice()运行时执行的具体步骤：

1. let A = new Array();

2. 将‘length’作为参数传递给[[Get]]內建方法，得到数组的长度len；

3. let relativeStart be [ToInteger](http://www.ecma-international.org/ecma-262/5.1/#sec-9.4)(start)；

4. 如果relativeStart < 0，那么actualStart = max((len + relativeStart), 0)。也就是说，**当第一个参数start为负时，实际上是从右往左开始计算的。**否则，actualStart = min(relativeStart, len)；

5. let actualDeleteCount = min(max([ToInteger]((http://www.ecma-international.org/ecma-262/5.1/#sec-9.4))(deleteCount), 0), len - actualStart)。这一步的作用和上一步是一样的，都是为了确保我们要操作的元素在一个合理的范围内，防止出现数组越界操作；

6. let k = 0；

7. Repeat, while(k < actualDeleteCount)

   ​	a. 获取数组在当前位置的元素值fromValue

   ​	b. 将fromValue添加到数组A中，具体实现上，是通过调用A的內建方法[DefineOwnProperty()](http://www.ecma-international.org/ecma-262/5.1/#sec-8.12.9)，这个方法有三个参数：ToString(k)、[Property Descriptor](http://www.ecma-international.org/ecma-262/5.1/#sec-8.10)、false。在[Property Descriptor](http://www.ecma-international.org/ecma-262/5.1/#sec-8.10)里，将fromValue设置为该属性的值，同时设置property的配置属性Writable：true , Enumerable: true, Configurable: true。用过Object.defineProperty()的同学应该对这些不陌生。

   ​	c. k++

   **这一步是为了将所有删除的元素添加到一个新的数组中，以便最后将其返回。**

8. 定义items为一个内部List，将item1及之后的所有参数（也就是我们要插入到数组中的元素）添加到items中。

9. 令itemCount为items为元素个数。

10. If itemCount < actualDeleteCount, then

  ​	a. let k = actualStart.

  ​	b. Repeat, while k < (len - actualDeleteCount)

  ​		i. let from = ToString(k + actualDeleteCount).

  ​	       ii. let to = ToString(k + itemCount).

  ​	      iii. 将from作为参数传递给內建的方法HasProperty，fromPresent为其返回的结果.

  ​	      iv. 如果fromPresent为true，获取数组在当前位置的元素值fromValue。然后调用数组內建方法[[Put]]，将to、fromValue、true作为参数传入。这样就实现了数组元素的移动。

  ​	       v.  如果fromPresent为false，调用数组的內建方法[[Delete]]，将to处的元素删除。

  ​	      vi. k++;

  ​	c. let k = len;

  ​	d. Repeat, while k > (len - actualDeleteCount + itemCount);

  ​		i. 调用数组內建方法[[Delete]]来删除k - 1位置上的元素。

  ​	       ii. k--

  ​	这里可能有点不太好理解，我们可以画图分析一下。splice()的运行可能出现三种情况：

  1. 只删除元素

  ![splice只删除元素](https://upload-images.jianshu.io/upload_images/192464-6871216f40be8f25.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

  ​	如果itemCount = 0且actualDeleteCount > 0，即不插入元素，只删除元素，那么from及其后面的所有元素会被移动到k位置后面，因此此时to = k。

  2. 只插入元素

     ![splice只插入元素](https://upload-images.jianshu.io/upload_images/192464-1bcd00b57fc04537.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

     如果itemCount > 0 且 actualDeleteCount = 0，即只插入元素的情况。此时from = k，即把k后面的所有元素移动到to后面，to = k + itemCount。

  3. 同时删除和插入元素

     ![splice同时插入和删除元素](https://upload-images.jianshu.io/upload_images/192464-bcf22c20af55c848.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

     如果itemCount > 0 且 actualDeleteCount > 0，即删除元素，也在插入元素。将from后面的元素移动到to后面。

     可以看到，这一步就是在插入元素数少于删除元素数时，进行元素的移动。

11. Else if itemCount > actualDeleteCount

    ​	a. let k = (len - actualDeleteCount).

    ​	b. Repeat, while k > actualStart

    ​		i. let from = ToString(k + actualDeleteCount - 1).

    ​	       ii. let to = ToString(k + itemCount - 1)

    ​	      iii. 将from作为参数传递给內建方法HasProperty，fromPresent为其返回的结果。

    ​	      iv. 如果fromPresent为true，获取数组在当前位置的元素值fromValue。然后调用数组內建方法[[Put]]，将to、fromValue、true作为参数传入。

    ​	       v. 如果fromPresent为false，调用数组的內建方法[[Delete]]，将to处的元素删除。

    ​	      vi. k--

12.  let k = actualStart.

13.  Repeat, while items is not empty.

     a. 删除items的一个元素E。

     b. 调用数组的內建方法[[Put]]，将k位置的元素更新为E。

     c. k++

    这一步实现了元素的插入。items实际上是一个队列，按照FIFO的顺序依次插入到actualStart后面。

14.  再次调用数组的內建方法[[Put]]来更新数组的‘length’属性，length = len - actualDeleteCount + itemCount.

15. 返回数组A。

以上就是splice()方法的工作原理。看到这里，你应该明白splice()效率不高的原因了吧。它的每次删除操作都涉及到大量元素的重新排列，而在插入元素时，引入了一个队列来管理。splice()只要涉及到删除操作，它的返回值都是一个包含所有删除元素的新数组。

在分析过程中，我对JavaScript中Array是对象这一概念也有了更深的了解。数组通过DefineOwnProperty()这一內建方法来为自己添加元素，它的Property Descriptor中Writable、Enumerable、Configurable等属性都为true，其实集合普通的对象属性是一样的，因此数组元素的值都可以被动态地修改。

另外，splice()的所有操作都是在原数组上进行的。JavaScript还提供了一个数组方法slice()，它不会修改原数组，函数式编程爱好者比较喜欢它。