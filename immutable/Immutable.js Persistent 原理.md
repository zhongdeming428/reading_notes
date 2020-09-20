## Immutable.js Persistent 原理

距离上一次写文章已经过去了五个月，现在终于决定继续沉下心来写一篇比较深入的文章。

这是一篇关于 Immutable.js 的文章，就像标题写的那样，不涉及 Immutable.js 的使用方式，只关注 Immutable.js 实现 persistent 的原理。

文章相关 ppt 和脑图已经上传到 GitHub：[reading_notes](https://github.com/zhongdeming428/reading_notes)，由于时间关系不会像以前那样在文章中进行过于详细的阐述，很多环节可能只讲述实现的基本原理，具体细节可以参考 ppt 和脑图或者源代码。

下面开始进入正题吧！

下图是本篇文章（也是对应 ppt）的内容提要：

![摘要](https://tva1.sinaimg.cn/large/007S8ZIlgy1gixadbs2kaj31140u0ag8.jpg)

### 一 基础介绍

#### 什么是 Immutable？

immutable 代表不可变的数据，即创建之后不可以再修改的数据。对应的有 mutable 数据，在创建之后仍然可以被修改。

#### JavaScript 中的 Immutable

JavaScript 中的所有原始数据类型（primitive value）都是 immutable 的；

JavaScript 中的所有复杂数据类型都是 mutable 的。

比如：

```js
const foo = 'string';
const bar = {};

let foo1 = foo;
const bar1 = bar;

foo1 = 'new-string';
bar1['CUSTOM_VALUE'] = [];

console.log(foo, bar, foo1, bar1);

// 输出：string {CUSTOM_VALUE: Array(0)} new-string {CUSTOM_VALUE: Array(0)}
```

由此可以发现，对于 foo1 的更改不会体现到 foo 上，而对于 bar1 的修改会影响到 bar。

这也可以解释为什么 string 类型可以通过 `[]` 索引字符，但是不可以对其赋值了。

#### Immutable & Mutable

产生 Immutable 和 Mutable 的根本原因在于数据存储方式的不同。对于简单的数据（可以理解为原始类型的数据），占用的存储空间小，操作简单，复制开销小，所以可以直接存储在栈内，每次赋值都可以直接进行值的复制；对于复杂的数据（可以理解为复杂类型的数据），占用的存储空间大，操作更加复杂，复制的时候开销比较大，所以存储在栈外的堆中，栈内只存储相应的指针。

我们在对复杂类型数据进行索引的时候，都是根据栈内存储的指针进行操作的，所以复制复杂类型数据的时候，实际上复制的也只是栈内的指针，两个指针会指向同一片内存区域，共享复杂类型的数据。这样做的好处是减少了操作复杂数据消耗的时间，降低了复制复杂数据所占用的多余空间。

两种类型的数据互有优劣，所以使用那种数据需要根据实际场景进行取舍，并不能说哪种数据就比哪种数据强。

二者的优劣对比如下：

|           | 优点                                                         | 缺点                                   |
| --------- | ------------------------------------------------------------ | -------------------------------------- |
| Mutable   | 充分利用内存，提高运行效率，减少了值复制和内存分配的时间。多数语言原生采用的方式，操作简便，较低学习成本。 | 数据操作更加危险，并发编程更加复杂。   |
| Immutable | 保证并发编程的安全性。 可以实现数据操作的原子性。            | 占用更多内存，操作效率较低，学习成本高 |

#### 为什么使用 Immutable？

上一小节说到需要针对不同的场景考量使用哪种类型的数据，所以这一小节就需要讲一讲为什么需要使用 Immutable 数据？

