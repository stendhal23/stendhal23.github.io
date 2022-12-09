- ⚛️ 通用的 GC 算法, 不限于语言 ⚛️

## 结论与事实

- 这类型算法的思路为: reclaim objects that are no longer *reachable* from a  *root set* .
  - root set 主要为函数栈上的 pointers.
- copying collector 需要两个内存区域, 一个称为 old space, 一个称为 new space
- 当 scan pointer 追上 allocate pointer 时, GC 完成.
- GC 完成: 意味着, 所有存活的对象已被复制到新区域, 且这些对象的中所有 pointer 类型的值皆已更新为新区域中的对象的地址。

## 过程

- 开始 GC 时, scan pointer 与 allocate pointer 都在 new space 的 base address, (即 offset 为 0)
- scan pointer :  scan pointer 位置的"后面" 是 "已 copy 完成 且 scan 完成的 object" ; scan pointer 的 "前面" 总是表示 "下一个已 copy 到 new space 但未 scan 的 object 的起始位置"
- allocate pointer : 每次 copy 某个 object 到 new space 时, allocate pointer 会 increment 该 object 所占的大小, 于是它总是表示 "下一次 copy object 时, 应该放在 new space 中的地址", 或者换个说法, 它的 "后面" 就是 "所有 已 copy 完成的 object"
- 一开始, 从 root set 开始, 检查该 set 中每一个 root, 如果它是 pointer 类型的, 则去 old space 中找到指向的 object,

```
// 伪代码 1
if(hasForwardPointer(oldSpaceObj))
{
	updatePointer(root, getForwardPointer(oldSpaceObj));
}
else
{
	newObjPtr = copyToNewSpace(oldSpaceObj);
	setForwardPointer(oldSpaceObj, newObjPtr);
}

```

- 直到 root set 全部检查完毕。 此时,  allocate pointer 已经 advance 到很远的位置, 但是 scan pointer 还没动, 依然在初始位置。
- 然后检查 scan pointer 指向的 "下一个已 copy 到 new space 但未 scan 的 object"
- scan 的过程:  检查该 object 的所有 component, 如果是 pointer 类型的, 则依然是用  "伪代码 1"  中的处理逻辑
  - scan 完成后, 我们相当于把 从该 object 出发,  immediately reachable 的 object 复制到 new space 或者 把 pointer component 更新为 new space 的 address 如果发现已经复制过的.
  - 每次 scan 只处理对当前 object 来说 immediately reachable 的 object,  至于这些 object 能 reach 的 object, 我们不考虑。因为这些更 "远" 的 object一定会被处理, 因为这些的 更 "远" 的 object 只是对当前 object 来说更远, 而它们却是 "当前 object 的 immediately reachable 的 object 的 immediately reachable 的 object".   所以后续 scan 时一定会处理它们。
  - scan 完成后, 该 object 的所有 pointer component 都能保证更新为最新的地址。
- 每 scan 完成一次, 则 scan pointer advance 到下一个未 scan 的 object 处。
- 当 scan pointer 追上 allocate pointer 时, 代表所有 reachable object 皆已 copy 到 new space, 且这些 object 中的所有 pointer component 皆已更新到指向 new space 中的 object.  即 GC 完成。
- GC 完成后, new space 就变成了下一次 GC 的 old space, 而 old space 则是作为下一次 GC 的 new space.  循环往复。
- 其他具体实现的改进:
  - 我们发现, 当 old space 与 new space 的大小是 1 : 1 时, 会造成内存的浪费: 比如如果每次 GC 存活的 object 大约为 20% 时,
    new space 就显得过大了。 所以比如 Java GC 就是做了一点改进, 分为 1 eden + 2 survior;  其实就相当于:
  - 第一轮 GC :
    old space = eden + survivor A
    new space  = survivor B
  - 第二轮 GC :
    old space = eden + survivor B
    new space  = survivor A
  - 循环往复.

```

```
