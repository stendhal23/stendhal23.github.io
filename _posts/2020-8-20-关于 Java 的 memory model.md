- ⚛️ 很多时候,我们知道一些分离的知识, 但是这些知识怎么联系起来呢? ⚛️
- ⚛️ 比如 data race, volatile, happens-before, MESI, 这些有什么联系吗? ⚛️

## 结论与事实

- Java 提供的 memory model 是 "有条件的顺序一致性"  (顺序一致性指术语 sequential consistency , 该概念有严格定义, 本文假设读者已经理解该定义)
- happens-before 是一个定义在 a set of operation 上的二元关系, 而且是个偏序关系.
- 是否存在 data race 是由 happens-before 关系来界定。

## 解释

- 下面涉及到的 quote 皆来自于 [JLS](https://docs.oracle.com/javase/specs/jls/se8/html/jls-17.html)
- 直接结论:  如果一段 java 程序没有 data race, 则 java 保证程序是 sequentially consistent

  > If a program has no data races, then all executions of the program will appear to be sequentially consistent.
  >
- 那什么叫 data race 呢? JLS 中是怎么定义的?

  - 答: 如果一个程序中, 某两个 conflicting accesses 之间没有 happens-before 关系, 则称这个程序包含一个 data race.

  > When a program contains two conflicting accesses ([§17.4.1](https://docs.oracle.com/javase/specs/jls/se8/html/jls-17.html#jls-17.4.1 "17.4.1. Shared Variables")) that are not ordered by a happens-before relationship, it is said to contain a *data race*.
  >
- 那什么是 conflicting access?

  - 答: 访问同一个 variable 的两个 access 行为, 且这两个行为中至少一个为 write, 则称两个 access  conflicting.

  > Two accesses to (reads of or writes to) the same variable are said to be *conflicting* if at least one of the accesses is a write.
  >
- 那 volatile 扮演了什么角色?

  - 答: 对一个 volatile variable 的 write access, 与 subsequent 的对同一 volatile variable 的 read 构成 happens-before 关系

  > A write to a `volatile` field ([§8.3.1.4](https://docs.oracle.com/javase/specs/jls/se8/html/jls-8.html#jls-8.3.1.4 "8.3.1.4. volatile Fields")) *happens-before* every subsequent read of that field.
  >

  - 上面这个 quote 就是 JLS 中关于 *happens-before 的定义中的一条.*
- 现在可以小结一下了:

  - volatile 能够  "消除 两个 conflicting access 的 data race 状态"   by   "使其不再满足 data race 的定义"
  - 更进一步地, 我们要使某段程序的多线程并发执行满足 sequential consistency, 即要求我们 "消除" 程序中的 data race, 即: 对 conflicting access 建立 happens-before 关系.  手段即是 happens-before 中的各种规定比如使用 monitor, 使用 volatile.
  - 而由于 happens-before 满足传递性, 而且 respect program order, 所以你能看到有些代码中有 if(volatile_flag) { // do something } 之类的. 因为这就是利用了 volatile 规则 + program order 规则 + 传递性, 为不同线程上的两个 access 之间建立了 happens-before 关系, 于是消除了 data race, 最终得到 sequential consistency 保证。
- 那么 MESI 之类的 cache coherence protocol 与这一切有什么关系吗?

  - 答案是没有关系。 有些人说什么 "利用 MESI 实现了 volatile " 之类的话, 都是不正确的。 是没理解 coherence 与 consistency 之间的差别造成的。
- coherence 与 consistency

  > Coherence deals with maintaining a global order in which writes to a single location or
  > single variable are seen by all processors. Consistency deals with the ordering of
  > operations to multiple locations with respect to all processors.
  >

  - 即, coherence 保证所有 cpu 对单个 register x, 要么统一是这个顺序:  write_x_1,  write_x_2, 要么统一是这个顺序: write_x_2,  write_x_1 ;
  - 而如果加入另一个 register y,  只有 coherence protocol 的情况下, 下面的顺序是可能的:
  - 对 cpu 0 而言, 有  write_x_1,   write_y_a,   write_x_2,  write_y_b;
  - 对 cpu 1 而言, 有  write_y_a,   write_x_1,   write_x_2,  write_y_b;
  - 而 consistency 是关于多个 register 的操作的 global order 的。即: 如果提供某种 consistency, 则 consistency 保证的是这样一个顺序在所有 cpu 上都是一样的:    write_x_1,  write_y_a;   比如上面那个就违反了 consistency 了。
