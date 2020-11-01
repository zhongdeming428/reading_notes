## Immutable.js Persistent 原理

距离上一次写文章已经过去了七个月，现在终于决定继续沉下心来写一篇比较深入的文章。

这是一篇关于 Immutable.js 的文章，就像标题写的那样，不涉及 Immutable.js 的使用方式，只关注 Immutable.js 实现 persistent 的原理。

文章相关 ppt 和脑图已经上传到 GitHub：[reading_notes](https://github.com/zhongdeming428/reading_notes)，由于时间关系不会像以前那样在文章中进行过于详细的阐述，很多环节可能只讲述实现的基本原理，具体细节可以参考 ppt 和脑图或者源代码。

下面开始进入正题吧！

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

Immutable 本来就是函数式编程中的一个概念，因为函数式编程强调数据的不可变性和函数的引用透明性，强调函数永远是一个不会修改外部数据的闭合功能单元。这样我们就可以单纯地思考当前函数的功能实现，在使用函数的时候也可以放心的调用它而不用担心它会影响到外部的或者全局的数据导致难以预测的问题。

所以使用 Immutable 数据的出发点就找到了，当我们进行函数式编程且需要尽可能地提高函数的引用透明性时，我们就应该使用 Immutable 类型的数据。这也是为什么 React 中需要使用 Immutable 而 Vue 并不 care。因为 Vue 开发并不遵循函数式编程范式，通过数据劫持和依赖收集可以自动解析到数据的变更，进而对 UI 进行 update。而 React 遵循函数式编程范式，其核心思想就是 `UI = f(state)`，所以我们编写的 React 组件越纯越好。

以上描述可能会比较抽象，所以下面举一个简单的例子：

![react example](https://tva1.sinaimg.cn/large/007S8ZIlgy1gixh3o5jlvj30u01ui10x.jpg)

上面有两个子组件和一个父组件，点击对应的 button 可以更新对应的子属性，但是点击 button 之后，可以看到另一个组件即使数据没有发生变化，也会进行 rerender。

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gixhjqjjtzj30u00vzjuo.jpg)

这时候我们就需要使用 React.memo 进行数据对比，向 React.memo 传入比对函数，比对前后数据是否发生变化，就可以阻止子组件在数据未发生变化的时候进行多余的 rerender。而使用 Immutable 数据，可以加快这个比对的过程，这就是为什么需要使用 Immutable 数据。另外在 React 中使用 immutable 还有一个好处就是当我们更改了数据之后，总是会返回一个新的数据对象，而不会在旧的数据上做修改，避免了直接更改旧的 state 而导致 React 无法接收到数据的变更消息。

#### 实际开发中的 Immutable 思想的应用

- 多数语言都已经自身或者通过第三方包实现 Immutable 数据结构。

  JavaScript/Elm/Python/Clojure/Java/...

- React 通过 Immutable 加快比对速度。

- 文件系统的 COW 机制，在复制数据到指定区间的时候，会先将旧的数据拷贝一份，避免意外导致的数据丢失。

### 二 实现方式

实现 Immutable 的方式有很多，下面介绍几种常见的方式。

#### 1 浅拷贝

通过浅拷贝我们就可以实现 Immutable 数据，这个想必大家都知道，因为 React 在官方文档中就介绍过这种方式：

```js
const a = {
  name: 'Russ',
  gender: 1,
  favoriteFoods: ['banana', 'watermelon', 'grape']
};

const b = {
  ...a,
  favoriteFoods: [
    ...a.favoriteFoods,
    'litchi'
  ]
};
```

通过这种方式，我们复用了 a 中的 `name` 和 `gender` 属性，以及 `a.favoriteFoods` 中的其他属性，并单独新增了一个 food。

这样就实现了 Immutable 数据，因为没有发生变更的数据都直接复用了，而且变更节点的祖先节点都变成了新的值。

但是这样的实现方式有几个弊端：

- 写法不优雅，当数据层级很深，数据结构复杂时，代码会变得很臃肿。
- 没有真正实现对原始类型数据的复用，实际上在进行扩展运算时，会重新创建新的原始类型数据。
- 对于数组而言，在首尾添加数据还好，但是要在指定索引修改值或者插入值的话，会变得更加复杂，需要对原始数组进行多次切割。
- 操作效率低下，对于数组而言，需要复制的节点数为 O(N)，索引越大，需要复制的节点数越多。

#### 2 链表

通过链表实现数据结构的共享大家可能也比较熟悉，因为和 git 的分支实现原理非常类似。通过使用不同的头指针指向不同的链表节点，可以实现数据结构的持久化：

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gixidk26qvj31ps0lkjt4.jpg)

如上图，我们如果要将 HEAD1 指向的节点 shift，只需要创建一个新的指针 HEAD2 指向第二个节点，这样就共享了 2 - 3 - 4 节点。

这时候如果我们要 unshift 一个新的节点，只需要创建一个新的节点，让新的头指针 HEAD3 指向它，然后将新的节点指向原来的 HEAD2 节点即可，这样又实现了 2 - 3 - 4 节点的复用。

如果我们要将 HEAD3 链表移除 2 节点，只需要复制 1‘ 节点，然后将其指向 3 节点，并且创建新的 HEAD4 头指针指向复制后的 1’‘ 节点。

所以通过链表实现 Immutable List，每次操作过程中可能需要复制的节点是被修改节点的左边所有节点，右边的节点都可以被共享。

但是这种实现方式的缺陷也是显而易见的：

- 存储效率低下（50%），每个节点只能存储一个数据和一个指针。
- 随机索引效率低下，需要从头遍历节点进行查找。

为了解决第一个缺陷，引入了一个新的数据结构：串。

#### 3 串

串是数组和链表的结合，将每个链表节点存储的数据从一个变成了多个。

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gixiq5sen3j31u0088mxr.jpg)

实现原理同链表，但是空间存储效率提高了。

但是缺陷仍然是随机索引效率较低。为了解决这个问题，我们考虑用平衡二叉树实现持久化数据结构。

#### 4 平衡二叉树

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gixisyx3ljj319e0lwn1g.jpg)

平衡二叉树左右节点均衡，高度差不会超过 1，所以针对不同长度的索引，操作/查找数据的时间复杂度以及需要复制的节点数非常稳定，均为 O(logN)。这样就大大提高了数据结构时间复杂度的稳定性，而且树的随机索引能力比链表强很多。

但是平衡二叉树也有自身的缺陷，就是空间存储效率极低，每个节点只能存储一个数据，但是需要存储两份指针。但是平衡二叉树给了我们很好的启发，通过树实现持久化数据结构，可以尽可能少且稳定的复制一定数量的节点。

结合数组和树，可以实现最适合持久化数据结构的 Vector Trie。

#### 5 Vector Trie

Vector Trie 也叫前缀树，是树结构的一种，它最为特殊的一点就是它的叶子节点只用来存储值，而非叶子节点都用来存储索引。

可能这么说会比较抽象，下面就画了一个前缀树的例子：

![](https://tva1.sinaimg.cn/large/0081Kckwgy1gk9ndvuk8hj31n00u0qbp.jpg)

左边是一个 JavaScript 对象，右边是对应的前缀树结构的实现。可以看到我们将键按字符拆分了，顺着 HEAD 指针出发挨个字符进行索引，就可以找到键对应的值。这就是一颗前缀树。

上面是 JavaScript 对象的实现，那么 JavaScript 数组是如何实现的呢？

我们知道 JavaScript 中有类数组对象的概念，那么我们可以用一个类数组对象来模拟数组的实现（暂时忽略 length 属性）：

![](https://tva1.sinaimg.cn/large/0081Kckwgy1gk9nmapwekj31nc0u0gug.jpg)

以上我们就用前缀树实现了 JavaScript 中的对象和数组，其实上面的第二种实现就是一颗标准的数字分区的前缀树，数字分区的意思就是用数字作为 key 对应到前缀树中。

下面是一颗更加标准化的十进制数字分区的前缀树：

![](https://tva1.sinaimg.cn/large/0081Kckwgy1gk9nuw9f75j31ty0oqjty.jpg)

这颗前缀树一共有 4 层，每个节点的子节点数为 10，所以整个前缀树可以存储的数据量是 10 ^ 4 = 10000。

上图表示了将这个前缀树作为一个 list，并将 list 的 2433 位置设置为 13 的过程，我们很快的就能够得到每层索引的计算公式如下：

```javascript
// levelIndex 是当前层级的索引
// level 是当前层级，从左至右依次为 4/3/2/1
const levelIndex = index / (10 ** (level - 1)) % 10
```

所以算出来结果就是 2/4/3/3，看起来就像把字符串 `2433` 按字符拆分一样的，所以很好理解。

与数字分区对应的是位分区，这个位就是 bit，当数字分区中每个节点的存储量是 2 的指数次方的时候，我们就得到了一颗位分区的前缀树，比如一颗最简单的位分区前缀树就是二进制的：

![](https://tva1.sinaimg.cn/large/0081Kckwgy1gk9ntgo0kij30wa0l20tl.jpg)

使用位分区有什么好处？最大的好处就是可以简化每层索引的计算公式，另外使用位分区可以更加符合计算机的构成原理以及语言的实现规则。

怎么理解这个好处呢，我们举个例子。

之前的十进制数字分区的每层索引计算公式为：

```js
// levelIndex 是当前层级的索引
// level 是当前层级
const levelIndex = index / (10 ** (level - 1)) % 10
```

如果我们的进制是 2 的指数次方，比如 2，那么上面的公式就可以简化为：

```js
// levelIndex 是当前层级的索引
// level 是当前层级
const levelIndex = index / (2 ** (level - 1)) % 2
```

但是针对二进制数据的运算，我们可以通过位运算简化，比如：

如果 `x = 2 ** n`，那么 `y / x = y >>> n`，`y % x = y & (x - 1)`。

所以当我们使用位分区的时候，每层索引运算公式就可以简化为：

```js
// levelIndex 是当前层级的索引
// level 是当前层级
const levelIndex = (index >>> (level - 1)) % 1
```

这就是使用位分区的最大好处，另外我们知道 JavaScript 中所有的数字都是以 64 位双精度的形式存储的，而且很多数据结构的最大长度都是 `2 ** 32 - 1`；所以使用位分区模拟 JavaScript 数据对象，会更加契合语言的设计，实现起来更加容易。

那前缀树怎么实现数据结构持久化呢？通过下面这张图我们就好理解了：

![](https://tva1.sinaimg.cn/large/0081Kckwgy1gk9ocfjdl5j31ay0u0dk9.jpg)

每次修改树的结构，我们都会拷贝索引路径上的所有节点，而旁系节点则维持原来的引用不变，这样就实现了数据结构的持久化。我们每次设置都可以得到一个全新的树结构，但是对于未修改的节点，引用未发生变化，所以看起来就实现了数据结构的共享。

综上所述，毫无疑问 Immutable.js 通过 32 位位分区实现了 List 结构及其数据结构持久化的特性。由于 JavaScript 数组一般情况下存储的最大长度为 `2 ** 32 - 1`，所以 Immutable.js 实现的 List 结构最多需要 7 层便可存储 `32 ** 7 = 2 ** 35` 个数据，满足了要求。而进行数据操作的时间复杂度为 O(7)。

了解了 Immutable.js 数据结构持久化的原理之后，我们其实也可以自己动手实现一下 List 结构；我已经写了一个基本实现，并且做了一些单元测试，代码扔在 [GitHub](https://github.com/zhongdeming428/immutable/tree/v1)，想看的同学可以去看看，这篇文章就不多讲了。

### 三 优化手段

#### 1 Tail 优化

之前提到的位分区前缀树已经能够满足我们的基本要求了，那么有没有需要优化的地方呢？

当然是有的，前面提到 Immutable.js 采用 32 位位分区前缀树实现 List 结构，所以操作节点的时间复杂度为 `O(7)`。虽然看起来是常数级复杂度，但是横向对比 JavaScript Native Array 来说，操作数据的速度还是差了几个量级，需要进行优化。

不优化时，每次修改数据都要进行多次计算以及节点的拷贝：

![](https://tva1.sinaimg.cn/large/0081Kckwgy1gk9p2sna0xg30bo04bb2b.gif)

而 List 的操作很多时候是在尾部进行的，所以我们可以针对这个特点进行一下优化，我们知道数据库/ DOM 都有批量插入的优化方案，所以我们这里也可以考虑使用批量修改的方案进行优化。

批量优化后的方案：

![](https://tva1.sinaimg.cn/large/0081Kckwgy1gk9ox7isi8g30bn06bkjn.gif)

实际上就是在 HEAD 指针上新加了一个 tail 属性，而 tail 的值就是前缀树中的一个普通节点，我们在尾部进行的操作都可以直接在 tail 节点上进行，当需要将 tail 节点挂载到树上时，再进行挂载，这样就可以节省很多时间。相比原来的方案，如果我们是 32 位位分区前缀树的实现，那么我们的所有尾部操作可以节省 31/32 的工作量，大大加快操作速度。

Tail 优化部分最重要的一点就是需要知道我们的操作的索引是不是在 tail 节点上，这里有一个计算公式，就是对于长度一定的 32 位位分区前缀树（假设长度为 capacity），那么 tail 开始的索引为：`(capacity - 1) >>> 5 << 5`，即 `(capacity - 1) / 32 * 32`。另外如果总体长度小于 32，那么 tail 开始的索引就是 0，因为只需要一个 tail 节点就可以完成整个 List 的实现。

Immutable.js 源码中的相关计算函数：

![](https://tva1.sinaimg.cn/large/0081Kckwgy1gk9pr2axi2j30wu0codhv.jpg)

#### 2 Transient 优化

在讲 Transient 优化之前，我们可以先看下面的代码有什么问题，这更有利于我们理解为什么需要 Transient 优化：

![](https://tva1.sinaimg.cn/large/0081Kckwgy1gk9pst33lrj30xy0kk77c.jpg)

上面的代码会重复对 `l` 进行赋值，以实现对索引 `0/1/2` 值的重写，咋一看也没什么问题，代码也是可以完美运行的。

但是通过前面的学习，我们知道每次调用 `set` 方法进行节点值设置的时候，我们都会得到一个全新的 List 实例，正如下图中的 `HEAD'` 和 `HEAD"`：

![](https://tva1.sinaimg.cn/large/0081Kckwgy1gk9py4ny73j31ud0u0q96.jpg)

整个过程中会不断的拷贝节点，新建节点，但是整个中间过程中的所有 List 实例我们都没有用到（比如 `HEAD'`），我们最终用到的只是最后一次调用 `set` 方法所返回的 List 实例（`HEAD"`），这会增加代码运行时的开销。所以我们要针对这一点进行优化。

优化的方案就是让中间生成的树变成 “mutable” 的，让我们可以直接修改其中的某些节点而不必每次都拷贝新建节点。

比如下图中的红色节点就是直接修改蓝色节点而产生的，我们整个过程中只产生了一颗新的 trie。

![](https://tva1.sinaimg.cn/large/0081Kckwgy1gk9q0kfif3j31q00u0q9f.jpg)

那我们怎么实现这种 `immutable` 过程中存在的 `mutable` 呢？关键点在于 `判断一个节点是否可以被修改的标准是什么？`。

通过对前缀树的理解，不难发现主要在于以下两个关键点：

1. 当前节点只能被当前 Trie 访问到，没有其他 Trie 共享当前节点（不能修改从其他 Trie 共享的节点）。
2. 当前 Trie 在本次更新后不会被用到（确保更新过程中没有产生新的 Trie）。

但是我们要如何实现这两个关键点的判断呢？

第一点还算好实现，我们可以通过 id 标识每个节点以及树的根节点，如果节点是在创建当前 trie 的时候新产生的，那么这个节点就应该和 trie 拥有相同的 id。

第二点比较难实现，怎么判断在本次更新之后不会被用到呢？这是很难的，谁也没办法预测未来会怎么。所以通过代码难以实现的目的，我们就只能通过规范来进行约束，规范可以是书面的，也可以是代码层面的，而 Immutable.js 就通过代码创建了这个规范。

看官网我们可以看到一个 `withMutations` 方法，就是实现这一优化方案的：

![](https://tva1.sinaimg.cn/large/0081Kckwgy1gk9qlgwkyjj31mk0u0k4k.jpg)

在传入的函数中，我们可以认为我们创建的 trie 都是不会在之后被使用的，可以直接对它进行修改，于是我们就不用在对同一个变量进行连续赋值，直接通过链式调用即可。

Immutable.js 中关于 Transient 优化的关键代码如下：

![](https://tva1.sinaimg.cn/large/0081Kckwgy1gk9qngy438j319c0lc791.jpg)

这个函数负责创建节点，接受两个参数：`node` 和 `ownerID`。其中 `node` 就是要修改的节点，`ownerID` 就是新产生的 `trie` 的 ID 标识。

如果当前的 `node` 和当前的 `trie` 拥有相同的 `ownerID`，那么我们可以认为这时候节点是 `mutable` 的，可以直接对它进行修改操作。

如果二者不相等，那么就表示该节点是从其他 `trie` 上共享的，所以不能对它进行修改。

而关于 `withMutations` 的实现如下：

```js
export function withMutations(fn) {
  // asMutable 会通过 ensureOwner 返回一个新的数据结构，并且这个数据结构一定会有一个 ownerID。
  const mutable = this.asMutable();
  fn(mutable);
  // 修改完之后这个 ownerID 需要还原成原来的 ownerID。
  return mutable.wasAltered() ? mutable.__ensureOwner(this.__ownerID) : this;
}
```

以上就是 `Transient` 优化的实现，这一实现对于内存空间的节省以及 GC 的优化并不明显，因为整个过程中回收的内存几近一致。更多的优化效果体现在执行速度上，因为不需要频繁的分配内存，新建对象，所以运行速度会快很多。下面是对 `1000000` 个元素的 `trie` 进行优化效果对比：

![](https://tva1.sinaimg.cn/large/0081Kckwgy1gk9qu97i3yj31oo0paqbq.jpg)

可以看到使用了 Transient 优化方式的代码执行速度快一倍。

关于这两个优化方案的实现，我也放在了 [GitHub](https://github.com/zhongdeming428/immutable)，相当于是 v2 版本，大家可以自己看代码。

关于我自己实现的 v1 和 v2 版本，还有 Immutable.js 的版本，我做了一次运行速度的比较：

![](https://tva1.sinaimg.cn/large/0081Kckwgy1gk9r0cl16cj31ic0u0h5y.jpg)

可以看到 Immutable.js 版本的运行速度比 v1 版本快了一倍，而 v2 版本的运行速度比 Immutable.js 版本的速度快了一倍。

前者比较好理解，后者不太好理解，下一节会解释为什么 v2 版本比 Immutable.js 版本还要快一倍。

#### 3 空间优化

通过 tail 优化和 Transient 优化，我们加快了代码的运行速度，但是还没有对存储的空间进行优化，这一节就讲如何对前缀树进行空间压缩。

上一节发现我们的 v2 版本比 Immutable.js 版本的运行速度要快一倍，个人认为是因为 Immutable.js 存在对于前缀树的压缩（可伸缩的前缀树），所以导致减慢了运行速度。而 v2 版本的实现是不可伸缩的前缀树，当我们创建一个 Trie 的时候，它就有了 `2 ** 25` 的容量，在之后的操作中，不再需要进行树空间的扩展，所以运行速度会相对快很多。

但是这样的弊端就是存储空间的浪费，我们可能只需要一个存储 5 容量的 Trie，但是却占用了 `2 ** 25` 个容量的 Trie 的空间，造成了资源的浪费，所以需要想办法进行优化，我们通过几组动画来看看 Immutable.js 是如何优化存储效率的。

__尾部有空间，直接进行 push__：

![](https://tva1.sinaimg.cn/large/0081Kckwgy1gk9rbvxliwg30bo080hdv.gif)

__尾部空间不足，但是 Trie 中空间足够时，新建所有节点__：

![](https://tva1.sinaimg.cn/large/0081Kckwgy1gk9rfzqf6pg30bo08lu11.gif)

__当整个 Trie 中都没有足够空间时，提升树的高度，使得容量提升一倍__：

![](https://tva1.sinaimg.cn/large/0081Kckwgy1gk9rhosqckg30bn04qb2b.gif)

__当容量足够时，直接 pop 节点__：

![](https://tva1.sinaimg.cn/large/0081Kckwgy1gk9s1en91bg30bo08aqv7.gif)

__当删除节点后，当前路径没有其他值时，删除整条路径，并且如果根节点上只存在一条路径时，可以对树进行压缩降级__：

![](https://tva1.sinaimg.cn/large/0081Kckwgy1gk9s5l28sfg30bn09c7wl.gif)



__通过游标实现元素的快速偏移，进而实现头部操作（shift/unshift）__：

![](https://tva1.sinaimg.cn/large/0081Kckwgy1gk9sd1tprwg30bn05pb2b.gif)

_每次对元素进行操作的时候，都需要计算当前的偏移量，而不是直接拿 index 进行计算。_

以上就是 Immutable.js 的空间优化方案，更多细节需要参考代码，链接在文末给出。

### 四 总结

由于时间原因，本篇文章更多的是讲实现原理，忽略了大部分的实现细节，需要读者在了解了原理之后再去看代码。

这里给出我总结的一些资料以及我参考的一些资料，方便大家进一步拓展深入。

#### 1 总结资料

强烈推荐大家仔细看看下方资料中的 PPT 和代码注解，因为内容比这篇文章更加翔实，还包括了对于 hash 算法/is 算法/Map 数据结构实现的讲解。

1. 学习过程中自己实现的 TrieList，代码地址：[immutable](https://github.com/zhongdeming428/immutable)，可以切换 v1 tag 查看 v1 版本
2. 学习过程中总结的脑图/PPT/文章：[reading_notes/immutable](https://github.com/zhongdeming428/reading_notes/tree/master/immutable)
3. 学习过程中对于源码的一些注解：[immutable-js](https://github.com/zhongdeming428/immutable-js)，注意是 master 分支

#### 2 参考资料

下面的文章非常推荐读者阅读，读完之后受益匪浅，绝对有助于提升你对 Immutable.js 实现的了解。

中文资料：

1. [Functional Go: 持久化数据结构简介](https://io-meter.com/2016/09/03/Functional-Go-persist-datastructure-intro/)
2. [Functional Go: Vector Trie 的实现](https://io-meter.com/2016/09/15/functional-go-implement-vector-trie/)
3. [Functional Go: Transient 及持久化](https://io-meter.com/2016/10/01/Functional-Go-Transient-and-Persistent/)
4. [深入探究Immutable.js的实现机制（一）](https://juejin.im/post/6844903679644958728)
5. [深入探究immutable.js的实现机制（二）](https://juejin.im/post/6844903682891333640)
6. [Immutable.js 源码解析 --List 类型](https://segmentfault.com/a/1190000017130003)
7. [Immutable.js 源码解析 --Map 类型](https://segmentfault.com/a/1190000017130413)
8. [读懂immutable-js中的Map数据结构](https://segmentfault.com/a/1190000016711478)
9. [JavaScript 引擎原理（三）](https://zhuanlan.zhihu.com/p/85836269)

外文资料：

1. [Persistent Vector Performance](https://hypirion.com/musings/persistent-vector-performance)
2. [Persistent Vector Performance Summarised](https://hypirion.com/musings/persistent-vector-performance-summarised)
3. [Understanding Clojure's Persistent Vectors, pt. 1](https://hypirion.com/musings/understanding-persistent-vector-pt-1)
4. [Understanding Clojure's Persistent Vectors, pt. 2](https://hypirion.com/musings/understanding-persistent-vector-pt-2)
5. [Understanding Clojure's Persistent Vectors, pt. 3](https://hypirion.com/musings/understanding-persistent-vector-pt-3)
6. [Understanding Clojure's Transients](https://hypirion.com/musings/understanding-clojure-transients)
7. [Data Structures in ImmutableJS](https://www.cheli.me/data-structures-in-immutablejs)
8. [Question: use of ``smi()`` function when computing hashes](https://github.com/immutable-js/immutable-js/issues/1704)
9. [Efficiently Representing Values and Tagging](https://github.com/thlorenz/v8-perf/blob/master/data-types.md#efficiently-representing-values-and-tagging)