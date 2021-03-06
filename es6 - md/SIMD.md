﻿# SIMD

标签： JavaScript ES6

---
##1.概述
SIMD（发音/sim-dee/）是“Single Instruction/Multiple Data”的缩写，意为“单指令，多数据”。它是JavaScript操作CPU对应指令的接口，你可以看做这是一种不同的运算执行模式。与它相对的是SISD（“Single Instruction/Single Data”），即“单指令，单数据”。

SIMD的含义是使用一个指令，完成多个数据的运算；SISD的含义是使用一个指令，完成单个数据的运算，这是JavaScript的默认运算模式。显而易见，SIMD的执行效率要高于SISD，所以被广泛用于3D图形运算、物理模拟等运算量超大的项目之中。

为了理解SIMD，请看下面的例子。
```js
var a = [1, 2, 3, 4];
var b = [5, 6, 7, 8];
var c = [];

c[0] = a[0] + b[0];
c[1] = a[1] + b[1];
c[2] = a[2] + b[2];
c[3] = a[3] + b[3];
c // Array[6, 8, 10, 12]
```
上面代码中，数组a和b的对应成员相加，结果放入数组c。它的运算模式是依次处理每个数组成员，一共有四个数组成员，所以需要运算四次。

如果采用SIMD模式，只要运算一次就够了。
```js
var a = SIMD.Float32x4(1, 2, 3, 4);
var b = SIMD.Float32x4(5, 6, 7, 8);
var c = SIMD.Float32x4.add(a, b); // Float32x4[6, 8, 10, 12]
```
上面代码之中，数组a和b的四个成员的各自相加，只用一条指令就完成了。因此，速度比上一种写法提高了4倍。

一次SIMD运算，可以处理多个数据，这些数据被称为“通道”（lane）。上面代码中，一次运算了四个数据，因此就是四个通道。

SIMD通常用于矢量运算。
```js
v + w    = 〈v1, …, vn〉+ 〈w1, …, wn〉
      = 〈v1+w1, …, vn+wn〉
```
上面代码中，v和w是两个多元矢量。它们的加运算，在SIMD下是一条指令、而不是n个指令完成的，这就大大提高了效率。这对于3D动画、图像处理、信号处理、数值处理、加密等运算是非常重要的。比如，Canvas的getImageData()会将图像文件读成一个二进制数组，SIMD就很适合对于这种数组的处理。

总的来说，SIMD是数据并行处理（parallelism）的一种手段，可以加速一些运算密集型操作的速度。将来无WebAssembly结合以后，可以让JavaScript达到二进制代码的运行速度。

---
##2.数据类型
SIMD提供12种数据类型，总长度都是128个二进制位。

 - Float32x4：四个32位浮点数
 - Float64x2：两个64位浮点数
 - Int32x4：四个32位整数
 - Int16x8：八个16位整数
 - Int8x16：十六个8位整数
 - Uint32x4：四个无符号的32位整数
 - Uint16x8：八个无符号的16位整数
 - Uint8x16：十六个无符号的8位整数
 - Bool32x4：四个32位布尔值
 - Bool16x8：八个16位布尔值
 - Bool8x16：十六个8位布尔值
 - Bool64x2：两个64位布尔值

每种数据类型被x符号分隔成两部分，后面的部分表示通道数，前面的部分表示每个通道的宽度和类型。比如，Float32x4就表示这个值有4个通道，每个通道是一个32位浮点数。

每个通道之中，可以放置四种数据。

 - 浮点数（float，比如1.0）
 - 带符号的整数（Int，比如-1）
 - 无符号的整数（Uint，比如1）
 - 布尔值（Bool，包含true和false两种值）

每种SIMD的数据类型都是一个函数方法，可以传入参数，生成对应的值。
```js
var a = SIMD.Float32x4(1.0, 2.0, 3.0, 4.0);
```
上面代码中，变量a就是一个128位、包含四个32位浮点数（即四个通道）的值。

注意，这些数据类型方法都不是构造函数，前面不能加new，否则会报错。
```js
var v = new SIMD.Float32x4(0, 1, 2, 3);
// TypeError: SIMD.Float32x4 is not a constructor
```

---
##3.静态方法：数学运算
每种数据类型都有一系列运算符，支持基本的数学运算。

---
###SIMD.%type%.abs()，SIMD.%type%.neg()
abs方法接受一个SIMD值作为参数，将它的每个通道都转成绝对值，作为一个新的SIMD值返回。
```js
var a = SIMD.Float32x4(-1, -2, 0, NaN);
SIMD.Float32x4.abs(a)
// Float32x4[1, 2, 0, NaN]
```
neg方法接受一个SIMD值作为参数，将它的每个通道都转成负值，作为一个新的SIMD值返回。
```js
var a = SIMD.Float32x4(-1, -2, 3, 0);
SIMD.Float32x4.neg(a)
// Float32x4[1, 2, -3, -0]

var b = SIMD.Float64x2(NaN, Infinity);
SIMD.Float64x2.neg(b)
// Float64x2[NaN, -Infinity]
```

---
###SIMD.%type%.add()，SIMD.%type%addSaturate()
add方法接受两个SIMD值作为参数，将它们的每个通道相加，作为一个新的SIMD值返回。
```js
var a = SIMD.Float32x4(1.0, 2.0, 3.0, 4.0);
var b = SIMD.Float32x4(5.0, 10.0, 15.0, 20.0);
var c = SIMD.Float32x4.add(a, b);
```
上面代码中，经过加法运算，新的SIMD值为(6.0, 12.0, 18.0, 24.0)。

addSaturate方法跟add方法的作用相同，都是两个通道相加，但是溢出的处理不一致。对于add方法，如果两个值相加发生溢出，溢出的二进制位会被丢弃；addSaturate方法则是返回该数据类型的最大值。
```js
var a = SIMD.Uint16x8(65533, 65534, 65535, 65535, 1, 1, 1, 1);
var b = SIMD.Uint16x8(1, 1, 1, 5000, 1, 1, 1, 1);
SIMD.Uint16x8.addSaturate(a, b);
// Uint16x8[65534, 65535, 65535, 65535, 2, 2, 2, 2]

var c = SIMD.Int16x8(32765, 32766, 32767, 32767, 1, 1, 1, 1);
var d = SIMD.Int16x8(1, 1, 1, 5000, 1, 1, 1, 1);
SIMD.Int16x8.addSaturate(c, d);
// Int16x8[32766, 32767, 32767, 32767, 2, 2, 2, 2]
```
上面代码中，Uint6的最大值是65535，Int16的最大值是32767.一旦发生溢出，就返回这两个值。

注意，Uint32x4和Int32x4这两种数据类型没有addSaturate方法。

---
###SIMD.%type%.sub(), SIMD.%type%.subSaturate()
sub方法接受两个SIMD值作为参数，将它们的每个通道相减，作为一个新的SIMD值返回。
```js
var a = SIMD.Float32x4(-1, -2, 3, 4);
var b = SIMD.Float32x4(3, 3, 3, 3);
SIMD.Float32x4.sub(a, b)
// Float32x4[-4, -5, 0, 1]
```
subSaturate方法跟sub方法的作用相同，都是两个通道相减，但是溢出的处理不一致。对于sub方法，如果两个值相减发生溢出，溢出的二进制位会被放弃；subSaturate方法则是返回该数据类型的最小值。
```js
var a = SIMD.Uint16x8(5, 1, 1, 1, 1, 1, 1, 1);
var b = SIMD.Uint16x8(10, 1, 1, 1, 1, 1, 1, 1);
SIMD.Uint16x8.subSaturate(a, b)
// Uint16x8[0, 0, 0, 0, 0, 0, 0, 0]

var c = SIMD.Int16x8(-100, 0, 0, 0, 0, 0, 0, 0);
var d = SIMD.Int16x8(32767, 0, 0, 0, 0, 0, 0, 0);
SIMD.Int16x8.subSaturate(c, d)
// Int16x8[-32768, 0, 0, 0, 0, 0, 0, 0, 0]
```
上面代码中，Uint16的最小值是0，Int16的最小值是-32678。一旦运算发生溢出，就返回最小值。

---
###SIMD.%type%.mul()，SIMD.%type%.div()，SIMD.%type%.sqrt()
mul方法接受两个SIMD值作为参数，将它们的每个通道相乘，作为一个新的SIMD值返回。
```js
var a = SIMD.Float32x4(-1, -2, 3, 4);
var b = SIMD.Float32x4(3, 3, 3, 3);
SIMD.Float32x4.mul(a, b)
// Float32x4[-3, -6, 9, 12]
```
div方法接受两个SIMD值作为参数，将它们的每个通道相除，作为一个新的SIMD值返回。
```js
var a = SIMD.Float32x4(2, 2, 2, 2);
var b = SIMD.Float32x4(4, 4, 4, 4);
SIMD.Float32x4.div(a, b)
// Float32x4[0.5, 0.5, 0.5, 0.5]
```
sqrt方法接受一个SIMD值作为参数，求出每个通道的平方根，作为一个新的SIMD值返回。
```js
var b = SIMD.Float64x2(4, 8);
SIMD.Float64x2.sqrt(b)
// Float64x2[2, 2.8284271247461903]
```

---
###SIMD.%FloatType%.reciprocalApproximation()，SIMD.%type%.reciprocalSqrtApproximation()
reciprocalApproximation方法接受一个SIMD值作为参数，求出每个通道的倒数（1/x），作为一个新的SIMD值返回。
```js
var a = SIMD.Float32x4(1, 2, 3, 4);
SIMD.Float32x4.reciprocalApproximation(a);
// Float32x4[1, 0.5, 0.3333333432674408, 0.25]
```
reciprocalSqrtApproximation方法接受一个SIMD值作为参数，求出每个通道的平方根的倒数（1/(x^0.5)），作为一个新的SIMD值返回。
```js
var a = SIMD.Float32x4(1, 2, 3, 4);
SIMD.Float32x4.reciprocalSqrtApproximation(a)
// Float32x4[1, 0.7071067690849304, 0.5773502588272095, 0.5]
```
注意，只有浮点数的数据类型才有这两个方法。

---
###SIMD.%IntegerType%.shiftLeftByScalar()
shiftLeftByScalar方法接受一个SIMD值作为参数，然后将每个通道的值左移指定的位数，作为一个新的SIMD值返回。
```js
var a = SIMD.Int32x4(1, 2, 4, 8);
SIMD.Int32x4.shiftLeftByScalar(a, 1);
// Int32x4[2, 4, 8, 16]
```
如果左移后，新的值超出了当前数据类型的位数，溢出的部分会被丢弃。
```js
var ix4 = SIMD.Int32x4(1, 2, 3, 4);
var jx4 = SIMD.Int32x4.shiftLeftByScalar(ix4, 32);
// Int32x4[0, 0, 0, 0]
```
注意，只有整数的数据类型才有这个方法。

---
###SIMD.%IntegerType%.shiftRightByScalar()
shiftRightByScalar方法接受一个SIMD值作为参数，然后将每个通道的值右移指定的位数，返回一个新的SIMD值。
```js
var a = SIMD.Int32x4(1, 2, 4, -8);
SIMD.Int32x4.shiftRightByScalar(a, 1);
// Int32x4[0, 1, 2, -4]
```
如果原来通道的值是带符号的值，则符号位保持不变，不受右移影响。如果是不带符号位的值，则右移后头部会补0。
```js
var a = SIMD.Uint32x4(1, 2, 4, -8);
SIMD.Uint32x4.shiftRightByScalar(a, 1);
// Uint32x4[0, 1, 2, 2147483644]
```
上面代码中，-8右移一位变成了2147483644，是因为对于32位无符号整数来说，-8的二进制形式是11111111111111111111111111111000，右移一位就变成了01111111111111111111111111111100，相当于2147483644。

注意，只有整数的数据类型才有这个方法。

---
##4.静态方法：通道处理

---
###SIMD.%type%.check()
check方法用于检查一个值是否为当前类型的SIMD值。如果是的，就返回这个值，否则就报错。
```js
var a = SIMD.Float32x4(1, 2, 3, 9);

SIMD.Float32x4.check(a);
// Float32x4[1, 2, 3, 9]

SIMD.Float32x4.check([1,2,3,4]) // 报错
SIMD.Int32x4.check(a) // 报错
SIMD.Int32x4.check('hello world') // 报错
```

---
###SIMD.%type%.extractLane()，SIMD.%type%.replaceLane()
extractLane方法用于返回给定通道的值。它接受；两个参数，分别是SIMD值和通道编号。
```js
var t = SIMD.Float32x4(1, 2, 3, 4);
SIMD.Float32x4.extractLane(t, 2) // 3
```
replaceLane方法用于替换指定通道的值，并返回一个新的SIMD值。它接受三个参数，分别是原来的SIMD值、通道编号和新的通道制。
```js
var t = SIMD.Float32x4(1, 2, 3, 4);
SIMD.Float32x4.replaceLane(t, 2, 42)
// Float32x4[1, 2, 42, 4]
```

---
###SIMD.%type%.load()
load方法用于从二进制数组读入数据，生成一个新的SIMD值。
```js
var a = new Int32Array([1,2,3,4,5,6,7,8]);
SIMD.Int32x4.load(a, 0);
// Int32x4[1, 2, 3, 4]

var b = new Int32Array([1,2,3,4,5,6,7,8]);
SIMD.Int32x4.load(a, 2);
// Int32x4[3, 4, 5, 6]
```
load方法接受两个参数：一个二进制数组和开始读取的位置（从0开始）。如果位置不合法（比如-1或者超出二进制数组的大小），就会抛出一个错误。

这个方法还有三个变种load1()、load2()、load3()，表示从指定位置开始，只加载一个通道、两个通道、三个通道的值。
```js
// 格式
SIMD.Int32x4.load(tarray, index)
SIMD.Int32x4.load1(tarray, index)
SIMD.Int32x4.load2(tarray, index)
SIMD.Int32x4.load3(tarray, index)

// 实例
var a = new Int32Array([1,2,3,4,5,6,7,8]);
SIMD.Int32x4.load1(a, 0);
// Int32x4[1, 0, 0, 0]
SIMD.Int32x4.load2(a, 0);
// Int32x4[1, 2, 0, 0]
SIMD.Int32x4.load3(a, 0);
// Int32x4[1, 2, 3,0]
```

---
###SIMD.%type%.store()
store方法用于将一个SIMD值，写入一个二进制数组。它接受三个参数，分别是二进制数组、开始写入的数组位置、SIMD值。它返回写入值以后的二进制数组。
```js
var t1 = new Int32Array(8);
var v1 = SIMD.Int32x4(1, 2, 3, 4);
SIMD.Int32x4.store(t1, 0, v1)
// Int32Array[1, 2, 3, 4, 0, 0, 0, 0]

var t2 = new Int32Array(8);
var v2 = SIMD.Int32x4(1, 2, 3, 4);
SIMD.Int32x4.store(t2, 2, v2)
// Int32Array[0, 0, 1, 2, 3, 4, 0, 0]
```
上面代码中，t1是一个二进制数组，v1是一个SIMD值，只有四个通道。所以写入t1以后，只有前四个位置有值，后四个位置都是0。而t2是从2号位置开始写入，所以前两个位置和后两个位置都是0。

这个方法还有三个变种store1()、store2()、store3()，表好似只写入一个通道、两个通道和三个通道的值。
```js
var tarray = new Int32Array(8);
var value = SIMD.Int32x4(1, 2, 3, 4);
SIMD.Int32x4.store1(tarray, 0, value);
// Int32Array[1, 0, 0, 0, 0, 0, 0, 0]
```

---
###SIMD.%type%.splat()
splat方法返回一个新的SIMD值，该值的所有通道都会设成一个预先给定的值。
```js
SIMD.Float32x4.splat(3);
// Float32x4[3, 3, 3, 3]
SIMD.Float64x2.splat(3);
// Float64x2[3, 3]
```
如果省略参数，所有整数型的SIMD值都会设定0，浮点型的SIMD值都会设成NaN。

---
###SIMD.%type%.swizzle()
swizzle方法返回一个新的SIMD值，重新排列原有的SIMD值的通道顺序。
```js
var t = SIMD.Float32x4(1, 2, 3, 4);
SIMD.Float32x4.swizzle(t, 1, 2, 0, 3);
// Float32x4[2,3,1,4]
```
上面代码中，swizzle方法的第一个参数是原有的SIMD值，后面的参数对应将要返回的SIMD值的四个通道。它的意思是新的SIMD的四个通道，依次是原来的SIMD值的1号通道、2号通道、0号通道、3号通道。由于SIMD值最多可以有16个通道，所以swizzle方法除了第一个参数以外，最多还可以接受16个参数。

下面是另一个例子。
```js
var a = SIMD.Float32x4(1.0, 2.0, 3.0, 4.0);
// Float32x4[1.0, 2.0, 3.0, 4.0]

var b = SIMD.Float32x4.swizzle(a, 0, 0, 1, 1);
// Float32x4[1.0, 1.0, 2.0, 2.0]

var c = SIMD.Float32x4.swizzle(a, 3, 3, 3, 3);
// Float32x4[4.0, 4.0, 4.0, 4.0]

var d = SIMD.Float32x4.swizzle(a, 3, 2, 1, 0);
// Float32x4[4.0, 3.0, 2.0, 1.0]
```

---
###SIMD.%tpye%.shuffle()
shuffle方法从两个SIMD值之中取出指定通道，返回一个新的SIMD值。
```js
var a = SIMD.Float32x4(1, 2, 3, 4);
var b = SIMD.Float32x4(5, 6, 7, 8);

SIMD.Float32x4.shuffle(a, b, 1, 5, 7, 2);
// Float32x4[2, 6, 8, 3]
```
上面代码中，a和b一共有8个通道，依次编号为0到7。shuffle根据编号，取出相应的通道，返回一个新的SIMD值。

---
##5.静态方法：比较运算
---
###SIMD.%type%.equal()，SIMD.%type%.notEqual()
equal方法用来比较两个SIMD值a和b的每一个通道，根据两者是否精确相等（a===b），得到一个布尔值。最后，所以通道的比较结果，组成一个新的SIMD值，作为掩码返回。notEqual方法则是比较两个通道是否不相等（a！==b）。
```js
var a = SIMD.Float32x4(1, 2, 3, 9);
var b = SIMD.Float32x4(1, 4, 7, 9);

SIMD.Float32x4.equal(a,b)
// Bool32x4[true, false, false, true]

SIMD.Float32x4.notEqual(a,b);
// Bool32x4[false, true, true, false]
```

---
###SIMD.%type%.greaterThan()，SIMD.%type%.greaterThanOrEqual()
greaThan方法用来比较两个SIMD值a和b的每一个通道，如果在该通道中，a较大就得到true，否则得到false。最后，所有通道的比较结果，组成一个新的SIMD值，作为掩码返回。greaterThanOrEqual则是比较a是否大于等于b。
```js
var a = SIMD.Float32x4(1, 6, 3, 11);
var b = SIMD.Float32x4(1, 4, 7, 9);

SIMD.Float32x4.greaterThan(a, b)
// Bool32x4[false, true, false, true]

SIMD.Float32x4.greaterThanOrEqual(a, b)
// Bool32x4[true, true, false, true]
```

---
###SIMD.%type%.lessThan()，SIMD.%type%.lessThanOrEqual()
lessThan方法用来比较两个SIMD值a和b的每一个通道，如果在该通道中，a较小就得到true，否则得到false。最后，所有通道的比较结果，会组成一个新的SIMD值，作为掩码返回。lessThanOrEqual方法则是比较a是否等于b。
```js
var a = SIMD.Float32x4(1, 2, 3, 11);
var b = SIMD.Float32x4(1, 4, 7, 9);

SIMD.Float32x4.lessThan(a, b)
// Bool32x4[false, true, true, false]

SIMD.Float32x4.lessThanOrEqual(a, b)
// Bool32x4[true, true, true, false]
```

---
###SIMD.%type%.select()
select方法通过掩码生成一个新的SIMD值。它接受三个参数，分别是也掩码和两个SIMD值。
```js
var a = SIMD.Float32x4(1, 2, 3, 4);
var b = SIMD.Float32x4(5, 6, 7, 8);

var mask = SIMD.Bool32x4(true, false, false, true);

SIMD.Float32x4.select(mask, a, b);
// Float32x4[1, 6, 7, 4]
```
上面代码中，select方法接受掩码和两个SIMD值作为参数。当某个通道对应的掩码为true时，会选择第一个SIMD值的对应通道，否则选择第二个SIMD值的对应通道。

这个方法通常与比较运算符结合使用。
```js
var a = SIMD.Float32x4(0, 12, 3, 4);
var b = SIMD.Float32x4(0, 6, 7, 50);

var mask = SIMD.Float32x4.lessThan(a,b);
// Bool32x4[false, false, true, true]

var result = SIMD.Float32x4.select(mask, a, b);
// Float32x4[0, 6, 3, 4]
```
上面代码中，先通过lessThan方法生成一个掩码，然后通过select方法生成一个由每个通道的较小值组成的新的SIMD值。

---
###SIMD.%BooleanType%.allType()，SIMD.%BooleanType%.anyTrue()
allTrue方法接受一个SIMD值作为参数，然后返回一个布尔值，表示该SIMD值的所有通道是否都为true。
```js
var a = SIMD.Bool32x4(true, true, true, true);
var b = SIMD.Bool32x4(true, false, true, true);

SIMD.Bool32x4.allTrue(a); // true
SIMD.Bool32x4.allTrue(b); // false
```
anyTrue方法则是只要有一个通道为true，就返回true，否则返回false。
```js
var a = SIMD.Bool32x4(false, false, false, false);
var b = SIMD.Bool32x4(false, false, true, false);

SIMD.Bool32x4.anyTrue(a); // false
SIMD.Bool32x4.anyTrue(b); // true
```
注意，只有四种布尔值数据类型（Bool32x4、Bool16x8、Bool8x16、Bool64x2）才有这两个方法。

这两个方法通常与比较运算符结合使用。
```js
var ax4    = SIMD.Float32x4(1.0, 2.0, 3.0, 4.0);
var bx4    = SIMD.Float32x4(0.0, 6.0, 7.0, 8.0);
var ix4    = SIMD.Float32x4.lessThan(ax4, bx4);
var b1     = SIMD.Int32x4.allTrue(ix4); // false
var b2     = SIMD.Int32x4.anyTrue(ix4); // true
```

---
###SIMD.%type%.min()，SIMD.%type%.minNum()
min方法接受两个SIMD值作为参数，将两者的对应通道的较小值，组成一个新的SIMD值返回。
```js
var a = SIMD.Float32x4(-1, -2, 3, 5.2);
var b = SIMD.Float32x4(0, -4, 6, 5.5);
SIMD.Float32x4.min(a, b);
// Float32x4[-1, -4, 3, 5.2]
```
如果有一个通道的值是NaN，则会优先返回NaN。
```js
var c = SIMD.Float64x2(NaN, Infinity)
var d = SIMD.Float64x2(1337, 42);
SIMD.Float64x2.min(c, d);
// Float64x2[NaN, 42]
```
minNum方法与min的作用一模一样，唯一的区别是如果有一个通道的值是NaN，则会有限返回另一个通道的值。
```js
var ax4 = SIMD.Float32x4(1.0, 2.0, NaN, NaN);
var bx4 = SIMD.Float32x4(2.0, 1.0, 3.0, NaN);
var cx4 = SIMD.Float32x4.min(ax4, bx4);
// Float32x4[1.0, 1.0, NaN, NaN]
var dx4 = SIMD.Float32x4.minNum(ax4, bx4);
// Float32x4[1.0, 1.0, 3.0, NaN]
```

---
###SIMD.%type%.max()，SIMD.%type%.maxNum()
max方法接受两个SIMD值作为参数，将两者的对应通道的较大值，组成一个新的SIMD值返回。
```js
var a = SIMD.Float32x4(-1, -2, 3, 5.2);
var b = SIMD.Float32x4(0, -4, 6, 5.5);
SIMD.Float32x4.max(a, b);
// Float32x4[0, -2, 6, 5.5]
```
如果有一个通道的值是NaN，则会优先返回NaN。
```js
var c = SIMD.Float64x2(NaN, Infinity)
var d = SIMD.Float64x2(1337, 42);
SIMD.Float64x2.max(c, d)
// Float64x2[NaN, Infinity]
```
maxNum方法与max的作用一模一样，唯一的区别是如果有一个通道的值是NaN，则会优先返回另一个通道的值。
```js
var c = SIMD.Float64x2(NaN, Infinity)
var d = SIMD.Float64x2(1337, 42);
SIMD.Float64x2.maxNum(c, d)
// Float64x2[1337, Infinity]
```

---
##6.静态方法：位运算
---
###SIMD.%type%.and()，SIMD.%type%.or()，SIMD.%type%.xor()，SIMD.%type%.not()
and方法接受两个SIMD值作为参数，返回两者对应的通道进行二进制AND运算（&）后得到的新的SIMD值。
```js
var a = SIMD.Int32x4(1, 2, 4, 8);
var b = SIMD.Int32x4(5, 5, 5, 5);
SIMD.Int32x4.and(a, b)
// Int32x4[1, 0, 4, 0]
```
上面代码中，以通道0为例，1的二进制形式是0001,5的二进制形式是01001，所以进行AND运算以后，得到0001。

or方法接受两个SIMD值作为参数，返回两者对应的通道进行二进制OR运算（|）后得到的新的SIMD值。
```js
var a = SIMD.Int32x4(1, 2, 4, 8);
var b = SIMD.Int32x4(5, 5, 5, 5);
SIMD.Int32x4.or(a, b)
// Int32x4[5, 7, 5, 13]
```
xor方法接受两个SIMD值作为参数，返回两者对应的通道进行二进制“疑惑”运算（^）后得到的新的SIMD值。
```js
var a = SIMD.Int32x4(1, 2, 4, 8);
var b = SIMD.Int32x4(5, 5, 5, 5);
SIMD.Int32x4.xor(a, b)
// Int32x4[4, 7, 1, 13]
```
not方法接受一个SIMD值作为参数，返回每个通道进行二进制“否”运算（~）后得到的新的SIMD值。
```js
var a = SIMD.Int32x4(1, 2, 4, 8);
SIMD.Int32x4.not(a)
// Int32x4[-2, -3, -5, -9]
```
上面代码中，1的否运算之所以得到-2，是因为在计算机内部，负数采用“2的补码”这种形式进行表示。也就是说，整数n的负数形式-n，是对每一个二进制位取反以后，再加上1。因此，直接取反就相当于负数形式再减去1，比如1的负数形式是-1，再减去1，就得到了-2。

---
##7.静态方法：数据类型转换
SIMD提供以下方法，用来将一种数据类型准尉另一种数据类型。

 - SIMD.%type%.fromFloat32x4()
 - SIMD.%type%.fromFloat32x4Bits()
 - SIMD.%type%.fromFloat64x2Bits()
 - SIMD.%type%.fromInt32x4()
 - SIMD.%type%.fromInt32x4Bits()
 - SIMD.%type%.fromInt16x8Bits()
 - SIMD.%type%.fromInt8x16Bits()
 - SIMD.%type%.fromUint32x4()
 - SIMD.%type%.fromUint32x4Bits()
 - SIMD.%type%.fromUint16x8Bits()
 - SIMD.%type%.fromUint8x16Bits()

带有Bits后缀的方法，会原封不动地将二进制位拷贝到新的数据类型；不带后缀的方法，则会进行数据类型转换。
```js
var t = SIMD.Float32x4(1.0, 2.0, 3.0, 4.0);
SIMD.Int32x4.fromFloat32x4(t);
// Int32x4[1, 2, 3, 4]

SIMD.Int32x4.fromFloat32x4Bits(t);
// Int32x4[1065353216, 1073741824, 1077936128, 1082130432]
```
上面代码中，fromFloat32x4是将浮点数转为整数，然后存入新的数据类型；fromFloat32x4Bits则是将二进制位原封不动地拷贝进入新的数据类型，然后进行解读。

Bits后缀的方法，还可以用于通道数目不对等的拷贝。
```js
var t = SIMD.Float32x4(1.0, 2.0, 3.0, 4.0);
SIMD.Int16x8.fromFloat32x4Bits(t);
// Int16x8[0, 16256, 0, 16384, 0, 16448, 0, 16512]
```
上面代码中，原始SIMD值t是4通道的，而目标值是8通道的。

如果数据转换时，原通道的数据大小，超过了目标通道的最大宽度，就会报错。

---
##8.实例方法
---
###SIMD.%type%.prototype.toString()
toString方法返回一个SIMD值的字符串形式。
```js
var a = SIMD.Float32x4(11, 22, 33, 44);
a.toString() // "SIMD.Float32x4(11, 22, 33, 44)"
```

---
##9.实例：求平均值
正常模式下，计算n个值的平均值，需要运算n次。
```js
function average(list) {
  var n = list.length;
  var sum = 0.0;
  for (var i = 0; i < n; i++) {
    sum += list[i];
  }
  return sum / n;
}
```
使用SIMD，可以将计算次数减少到n次的四分之一。
```js
function average(list) {
  var n = list.length;
  var sum = SIMD.Float32x4.splat(0.0);
  for (var i = 0; i < n; i += 4) {
    sum = SIMD.Float32x4.add(
      sum,
      SIMD.Float32x4.load(list, i)
    );
  }
  var total = SIMD.Float32x4.extractLane(sum, 0) +
              SIMD.Float32x4.extractLane(sum, 1) +
              SIMD.Float32x4.extractLane(sum, 2) +
              SIMD.Float32x4.extractLane(sum, 3);
  return total / n;
}
```
上面代码是每隔四位，将所有的值读入一个SIMD，然后立刻累加，得到累加值四个通道的总和，再除以n就可以了。