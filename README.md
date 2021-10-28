#! https://zhuanlan.zhihu.com/p/426966480

# 用 TypeScript 类型运算实现一个中国象棋程序

众所周知，TypeScript 是图灵完备的，因此，只要我们愿意，那当然是可以用它来实现一个象棋程序的。于是我们就快乐地开始了，为了理解方便，我们不考虑性能优化策略，纯粹从功能实现角度去构建。另外，我们尝试用中文来编写这个程序，因为类型运算中需要用到的操作符很少，类型本质上是一种对现实世界的描述，某种程度上算是一种业务描述语言，使用中文也挺好的。

## 需求拆解

首先我们要先来分析一下，需要实现哪些功能。

象棋的核心功能就是走棋，走棋如果用一个函数来表达，那就是：`走棋<当前棋局, 源位置, 目标位置>`，因此，这里就涉及两个要素：

- 棋局结构：棋子行、列的组织
  - 棋子结构：类型、颜色
  - 位置：横坐标，纵坐标
- 棋子的移动
  - 这个棋子能放在这里吗？
  - 这个棋子能这么移动吗？

比较多的细节会出现在棋子的移动规则上，但是，不重要，都是细节，我们挨个来解释吧。

## 基础结构

### 棋子

棋子的话，比较简单，没有什么特别的，只要表达出颜色和种类就可以了：

```ts
type 棋子 = {
  颜色: 棋子颜色
  种类: 棋子种类
}

type 构造棋子<某颜色, 某种类> = 某颜色 extends 棋子颜色
  ? 某种类 extends 棋子种类
    ? {
        颜色: 某颜色
        种类: 某种类
      }
    : 不可能
  : 不可能
```

`本节使用的技巧非常简单，就是一个合法性判断而已。`

### 位置

位置就会复杂不少了，主要原因是，在类型系统里面，我们要表示整数及其运算是一件比较麻烦的事情，而偏偏在计算走棋规则的时候，这部分很重要，所以，不得不去做。

定义整数和加减法的部分，在其他文章里可以找到，不在这里叙述了，假定已经存在了一些常用的整数变量，可以得到对于位置信息的表达：

```ts
type 棋子横坐标 = 零 | 一 | 二 | 三 | 四 | 五 | 六 | 七 | 八
type 棋子纵坐标 = 零 | 一 | 二 | 三 | 四 | 五 | 六 | 七 | 八 | 九

type 最大横坐标 = 八
type 最大纵坐标 = 九

type 棋子坐标 = {
  横: 棋子横坐标
  纵: 棋子纵坐标
}

type 构造棋子坐标<横坐标, 纵坐标> = 横坐标 extends 棋子横坐标
  ? 纵坐标 extends 棋子纵坐标
    ? {
        横: 横坐标
        纵: 纵坐标
      }
    : 不可能
  : 不可能
```

位置比棋子要复杂的原因是，要支持一些运算，比如，判断两个位置是否相同

```ts
type 相同位置<位置1, 位置2> = 位置1 extends 棋子坐标
  ? 位置2 extends 棋子坐标
    ? 相等<位置1['横'], 位置2['横']> & 相等<位置1['纵'], 位置2['纵']>
    : 不可能
  : 不可能
```

然后，我们又会需要，对一个位置进行移动，用来定义寻找相对位置的快捷操作，比如，上下左右相邻位置的坐标，需要注意的是，这里要做一下合法性校验，避免越界。

```ts
type 左邻位<位置> = 位置 extends 棋子坐标
  ? 大于<位置['横'], 零> & 构造棋子坐标<减一<位置['横']>, 位置['纵']>
  : 不可能

type 右邻位<位置> = 位置 extends 棋子坐标
  ? 小于<位置['横'], 最大横坐标> & 构造棋子坐标<加一<位置['横']>, 位置['纵']>
  : 不可能

type 上邻位<位置> = 位置 extends 棋子坐标
  ? 大于<位置['纵'], 零> & 构造棋子坐标<位置['横'], 减一<位置['纵']>>
  : 不可能

type 下邻位<位置> = 位置 extends 棋子坐标
  ? 小于<位置['纵'], 最大纵坐标> & 构造棋子坐标<位置['横'], 加一<位置['纵']>>
  : 不可能
```

`本节使用的规则稍稍复杂一些，但主要复杂度不在这里，而是在定义数字运算的地方。`

## 棋局

在棋局这里，我们第一次面对了复杂的状况，整个这部分，需要解决这么一些问题：

- 棋子的行列结构如何构造？
- 如何基于某个位置的横纵坐标进行棋子的获取？
- 如何为棋子规则提供必要的帮助？
- 如何移动棋局中的棋子？

我们逐一来解决这些问题。

### 棋局结构

棋局结构的建模，是与取值操作密切相关的，需要注意到的是，我们并没有使用数字字面量去代表数字，而是自定义了整数类型，这就意味着，取值操作的索引是很复杂的，它无法直接得到。

在我们使用的整数定义中，数字是用一级一级嵌套表达的，当我们表示数字`五`，其等价形态为：

```ts
type 整数 = 零 | { 前一个数: 整数; 是零吗: 否 }

type 五 = 加一<加一<加一<加一<加一<零>>>>>
```

当我们得到一个数字，并且想要知道它真正表达的是多少，需要去统计从它到零的距离，也就是说，要反向去根据`是零吗`和`前一个数`这两个类型属性，一直往前找到零的位置。

有鉴于此，我们选择把棋局也构建为类似形式：

- 棋局的内容是棋局行形成的链表，棋局持有第零行
- 每行的内容是单元格形成的链表，棋局行持有第零格

```ts
type 构造棋局单元<内容参数, 下一个> = 内容参数 extends 单元格内容参数
  ? 下一个 extends 棋局单元 | 空
    ? {
        内容: 内容参数
        下一个: 下一个
      }
    : 不可能
  : 不可能

type 构造棋局行<内容参数, 下一行> = 内容参数 extends 行内容参数
  ? 下一行 extends 棋局行 | 空
    ? 内容参数 extends [
        infer 第零格,
        infer 第一格,
        infer 第二格,
        infer 第三格,
        infer 第四格,
        infer 第五格,
        infer 第六格,
        infer 第七格,
        infer 第八格
      ]
      ? {
          内容: 构造棋局单元<
            第零格,
            构造棋局单元<
              第一格,
              构造棋局单元<
                第二格,
                构造棋局单元<
                  第三格,
                  构造棋局单元<
                    第四格,
                    构造棋局单元<
                      第五格,
                      构造棋局单元<
                        第六格,
                        构造棋局单元<第七格, 构造棋局单元<第八格, 空>>
                      >
                    >
                  >
                >
              >
            >
          >
          下一行: 下一行
        }
      : 不可能
    : 不可能
  : 不可能

type 构造棋局<内容 extends 棋局内容参数 = 棋局内容参数> = 内容 extends [
  infer 第零行,
  infer 第一行,
  infer 第二行,
  infer 第三行,
  infer 第四行,
  infer 第五行,
  infer 第六行,
  infer 第七行,
  infer 第八行,
  infer 第九行
]
  ? {
      内容: 构造棋局行<
        第零行,
        构造棋局行<
          第一行,
          构造棋局行<
            第二行,
            构造棋局行<
              第三行,
              构造棋局行<
                第四行,
                构造棋局行<
                  第五行,
                  构造棋局行<
                    第六行,
                    构造棋局行<
                      第七行,
                      构造棋局行<第八行, 构造棋局行<第九行, 空>>
                    >
                  >
                >
              >
            >
          >
        >
      >
    }
  : 不可能
```

这个看上去非常诡异，但是我们后续对它的取值就很简单了。

`本节主要使用了一个 infer 来定义临时变量，然后就是递归去构造了（这里可能是有简便写法的，不过我暂时想不到）。`

### 棋局的取值

这一步，我们的目标是实现根据给定位置，从棋局中获取对应的棋子。

先来考虑如何根据一个行号获取指定行，其实很简单，让行号与棋局行链表同步移动就可以了，行号每次减一，行链表每次后移一行，那么，当行号减小到零的时候，行链表指向的就是所要的那行了：

```ts
type 根据链表获取棋局指定行<
  行链表 extends 棋局行 | 空,
  行号 extends 棋子纵坐标
> = 行链表 extends 棋局行
  ? 是 extends 相等<行号, 零>
    ? 行链表
    : 根据链表获取棋局指定行<行链表['下一行'], 减一<行号>>
  : 不可能
```

同理，在一行中获取某一列，也是一样的方式：

```ts
type 根据链表获取棋局某行的指定单元格<
  单元链表 extends 棋局单元 | 空,
  列号 extends 棋子横坐标
> = 单元链表 extends 棋局单元
  ? 是 extends 相等<列号, 零>
    ? 单元链表
    : 根据链表获取棋局某行的指定单元格<单元链表['下一个'], 减一<列号>>
  : 不可能
```

我们把这两者组合起来，即可得到根据位置查找棋子的类型函数：

```ts
type 获取棋局某位置的单元<
  某棋局 extends 棋局,
  某位置 extends 棋子坐标
> = 获取某行的指定单元格<获取棋局指定行<某棋局, 某位置['纵']>, 某位置['横']>
```

`本节的主要技巧是类型递归，获取行、单元格的时候，都是递归调用当前的类型函数自身去取值的。`

### 辅助规则的提供

在计算棋子位置关系的时候，我们会需要一些判断规则，这些规则的实现，都可以通过组合现有函数的方式去实现。

一些简单规则我们就不去探讨了，比如说，马的“日”字顶角，象的“田”字顶角，还有马脚，象眼等等，都可以反复组合之前我们做好的计算邻位的四个函数，比如：

```ts
type 象可以走到右上的目标位置<当前位置> = 右邻位<
  右邻位<上邻位<上邻位<当前位置>>>
>
```

我们重点来看一个相对复杂的：统计一条直线上两个棋子之间的棋子数量。这里的麻烦在于，统计个数需要有一个累加计数功能，有趣的是，它仍然可以利用递归。

```ts
/**
 * 当需要寻找一条线上两点之间有几个棋子的时候，步骤是：
 * - 如果两点重合，返回零
 * - 否则沿着起点方向逐步向终点靠近，如果下一个点不是终点，并且有棋子，则累加一
 */
type 同一水平线上两个位置之间有几个棋子<
  纵坐标,
  横坐标1,
  横坐标2,
  当前棋局 extends 棋局,
  初始值 extends 整数
> = 将两个整数从小到大排序<[横坐标1, 横坐标2]> extends [infer 小, infer 大]
  ? 小 extends 棋子横坐标
    ? 大 extends 棋子横坐标
      ? 是 extends 相等<小, 大>
        ? 初始值
        : 同一水平线上两个位置之间有几个棋子<
            纵坐标,
            加一<小>,
            大,
            当前棋局,
            是 extends 相等<加一<小>, 大>
              ? 初始值 // 下一个就是终点了，中间没有棋子可以累加了
              : 是 extends 该位置有棋子<
                  当前棋局,
                  构造棋子坐标<加一<小>, 纵坐标>
                >
              ? 加一<初始值>
              : 初始值
          >
      : 不可能
    : 不可能
  : 不可能
```

`本节使用了一个小技巧，先把两个坐标排序了一下。这样后续的递归方向可以不需要判断，否则要考虑两个方向。`

### 棋子的移动

棋子的移动，本质上很简单，就是把目标位置的棋子内容替换掉就可以了，但是，在 TypeScript 的类型系统里，“替换”是个不太可行的事情，它是 immutable 的，所谓的替换，实际上是创建了新的。

这样，我们就得实现棋局每个层级的拷贝构造操作了，这也不难：

```ts
type 从内容构造棋局单元<
  内容参数 extends 单元格内容参数,
  下一个 extends 棋局单元 | 空
> = {
  内容: 内容参数
  下一个: 下一个
}

type 从内容构造棋局行<单元链表 extends 棋局单元, 下一行 extends 棋局行 | 空> = {
  内容: 单元链表
  下一行: 下一行
}

type 从内容构造棋局<行链表 extends 棋局行> = {
  内容: 行链表
}
```

然后，真正复杂的是替换过程中的重新构造，以替换棋局中的某一行为例，这实际上是一个一边遍历，一边根据需要构造的过程，本质上是一个 copy on write 结构，理解了这点之后，写起来就不难了，仍然是递归：

```ts
type 将棋局的某行替换为指定行<
  某棋局 extends 棋局,
  行号 extends 棋子纵坐标,
  新内容 extends 棋局单元,
  当前迭代行号 extends 整数
> = 当前迭代行号 extends 棋子纵坐标
  ? 根据链表获取棋局指定行<
      某棋局['内容'],
      当前迭代行号
    > extends infer 原先的这行
    ? 原先的这行 extends 棋局行
      ? 是 extends 相等<行号, 当前迭代行号>
        ? 从内容构造棋局行<新内容, 原先的这行['下一行']> // 如果是要替换的这行，重新构造整个行，把原来的后续单元挂接过来就行了
        : 从内容构造棋局行<
            原先的这行['内容'],
            将棋局的某行替换为指定行<某棋局, 行号, 新内容, 加一<当前迭代行号>>
          >
      : 不可能
    : 不可能
  : 空
```

同理，替换单元格也可以写出来：

```ts
type 将棋局行的某单元替换为指定单元<
  某行 extends 棋局行,
  列号 extends 棋子横坐标,
  新内容 extends 单元格内容参数,
  当前迭代列号 extends 整数
> = 当前迭代列号 extends 棋子横坐标
  ? 根据链表获取棋局某行的指定单元格<
      某行['内容'],
      当前迭代列号
    > extends infer 原先的这个单元
    ? 原先的这个单元 extends 棋局单元
      ? 是 extends 相等<列号, 当前迭代列号>
        ? 从内容构造棋局单元<新内容, 原先的这个单元['下一个']> // 如果是要替换的单元格，重新构造这个单元格，把原来的后续单元挂接过来就行了
        : 从内容构造棋局单元<
            原先的这个单元['内容'],
            将棋局行的某单元替换为指定单元<
              某行,
              列号,
              新内容,
              加一<当前迭代列号>
            >
          >
      : 不可能
    : 不可能
  : 空
```

然后组合两者，就可以得到在棋局上指定位置替换棋子的函数：

```ts
type 将棋局某位置替换为指定棋子<
  某棋局 extends 棋局,
  某位置 extends 棋子坐标,
  某棋子或者空 extends 单元格内容参数
> = 获取棋局某位置的单元<某棋局, 某位置> extends infer 原位置的单元
  ? 原位置的单元 extends 棋局单元
    ? 获取棋局指定行<某棋局, 某位置['纵']> extends infer 原有的行
      ? 原有的行 extends 棋局行
        ? 从内容构造棋局<
            将棋局的某行替换为指定行<
              某棋局,
              某位置['纵'],
              将棋局行的某单元替换为指定单元<
                原有的行,
                某位置['横'],
                某棋子或者空,
                零
              >,
              零
            >
          >
        : 不可能
      : 不可能
    : 不可能
  : 不可能
```

两次使用这个替换，即可得到走棋的函数：

```ts
type 走棋<
  当前棋局 extends 棋局,
  起始位置 extends 棋子坐标,
  目标位置 extends 棋子坐标
> = 获取棋局某位置的棋子<当前棋局, 起始位置> extends infer 待移动的棋子
  ? 待移动的棋子 extends 棋子
    ? 是 extends 棋子可以从这里走到那里吗<起始位置, 目标位置, 当前棋局>
      ? 将棋局某位置替换为指定棋子<
          将棋局某位置替换为指定棋子<当前棋局, 起始位置, 空>,
          目标位置,
          待移动的棋子
        >
      : 不可能
    : 不可能
  : 不可能
```

`本节没有出现新的技巧，只是因为之前的链表结构实现，导致复制起来相对复杂一些。`

### 棋局的渲染

当我们建立了棋局的走棋规则之后，实际上象棋已经可用了，但为了显示的友好，还需要多做一步。

最基础的渲染单元，是单个的单元格，这个非常简单，直接根据颜色和种类进行两级映射就可以了。

```ts
type 渲染棋子<某个棋子> = 某个棋子 extends 棋子
  ? {
      红: {
        将: '帅'
        士: '仕'
        象: '相'
        马: '傌'
        车: '俥'
        炮: '炮'
        兵: '兵'
      }
      黑: {
        将: '将'
        士: '士'
        象: '象'
        马: '马'
        车: '车'
        炮: '炮'
        兵: '卒'
      }
    }[某个棋子['颜色']][某个棋子['种类']]
  : 不可能

type 渲染单元格<某单元格> = 某单元格 extends 棋子 ? 渲染棋子<某单元格> : '➕'
```

稍微复杂一些的是渲染行，考虑到我们之前把棋局行实现成了单元格链表，这个就需要一个递归了。

```ts
type 渲染单元链表<
  某单元 extends 棋局单元,
  初始渲染结果 extends string
> = 某单元['下一个'] extends 棋局单元
  ? 渲染单元链表<
      某单元['下一个'],
      `${初始渲染结果} ${渲染单元格<某单元['内容']>}`
    >
  : `${初始渲染结果} ${渲染单元格<某单元['内容']>}`
```

接下来，我们需要把每行的渲染结果合并输出，由于在 TypeScript 中，不管是字符串还是数组，都无法控制在 IDE 提示中的换行，因此，我们需要把结果渲染成对象结构。又因为通过字符串形态的整数，无法关联到真实的整数类型，所以需要建立一个辅助索引。

```ts
type 数字键值对 = {
  零: 零
  一: 一
  二: 二
  三: 三
  四: 四
  五: 五
  六: 六
  七: 七
  八: 八
  九: 九
}

type 渲染棋局<某个棋局 extends 棋局> = {
  [key in
    | '零'
    | '一'
    | '二'
    | '三'
    | '四'
    | '河'
    | '五'
    | '六'
    | '七'
    | '八'
    | '九']: key extends '河'
    ? '┠ ～ 楚 河 ～ ～ ～ 汉 界 ～ ┨'
    : 渲染指定行<
        某个棋局,
        key extends keyof 数字键值对 ? 数字键值对[key] : 不可能
      >
}
```

这样，整个棋局就可以被渲染出来了。

`本节使用的技巧主要是字符串的拼接，以及对象结构的输出。`

## 小结

本文的意图主要是在不考虑优化的情况下，如何借助类型系统的运算来解构这么一个象棋逻辑。

主体功能都还是比较容易实现的，从目前的实现来看，调用栈太深了，现在走棋已经很难跑出来了，需要优化，比如把数字的计算改成打表，可以优化不少，暂时先不去做它了。

特别鸣谢在此文编写过程中提供帮助的`引证`老师。
