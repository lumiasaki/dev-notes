## 一种对 Timer 的复用策略

### 场景：

例如很常见的大促列表页，每一个 `Cell` 都有活动结束倒计时。

### 不那么好的实现

每个需要用到 `timer` 的地方都 `new Timer()`，该 `timer` 到时间后会调用指定的 `callback`，需求完成。

#### 不好在哪里

根据 `Foundation` 的实现可知，`NSTimer` 依赖于 `Runloop`，将其作为一种 `source` 注册到 `Runloop` 的 `observers` 中，而在这个过程中没有复用。这样的后果是，其实我们并不需要那么多 `Timer`，因为这些 `timer` 的回调时间其实是相同的（对于之前提到的大促列表中的倒计时场景），或者说有一类是有相同的触发时间，对资源是一种浪费，也增加了维护 `timers` 的成本。

### 一种改进的思路

通过复用 `timer` 的方式，将具有同样回调时间的那些 `timer` 申请合并成同一个 `timer`，底层共享一个 `timer`，该 `timer` 可以用 `NSTimer`，也可以使用其他的实现来提高精度，这里将细节隐藏，调用者不需要关心。

#### 怎么设计？

首先，基本的数据结构是一个 `Map`。

该 `Map` 的 `key` 为 `timeout` 时间，而 `value` 为一个链表和一个真实的 `timer`。

结构大致可以如下：

```javascript
{
  '5': {
    timer: an instance of NSTimer / CADisplayLink / dispatch_source_t / ... ,
    linkedAppliers: [{
      applier,
      applyTime,
      timeout,
      repeat,
      callback
    }, ...]
  },
  ...
}
```

通过用同一个key，将那些同样回调时间的聚合在一个 `value` 上。

`value` 中的 `linkedAppliers` 是一个链表，外部每申请一个 `timer` 时，就将一个对象以 `O(1)` 的复杂度 `append` 到这个链表的最后，这个链表的每一项，保存如下信息：

1. 谁申请的（弱引用）
2. 申请的时间（因为复用了 `timer`，但链表中申请的对象们并不是同时申请的，因而它们的回调触发时间也是不同的，触发时间应该为`申请时间 + 过期时间`）
3. 过期时间（冗余存）
4. repeat（标记是否在触发一次后继续循环触发）
5. callback（触发后调用的回调）

这样，在同一个key下，`linkedAppliers`链表中靠前的申请，总会比靠后的申请更早触发。每次该 `timeout` 时间的 `timer`（即顶层对象中的 `timer` 对象持有的真实 `timer`）被触发时，检查当前时间是否晚于该链表头部的一个申请的过期时间，如果晚了的话，则触发该申请的 `callback`，同时将其从该链表头部删除。接下来检查其 `repeat` 标记，如果为`true`，则重新将该申请 `append` 到链表的末尾；如果为`false`，则该申请的周期结束。

当一个链表中的所有申请均被执行后，也即`linkedAppliers`链表为空时，对顶层对象中该 `timeout` 时间的 `timer` 执行`invalidate()`和置`nil`操作，并从顶层对象中将该 `timeout` 时间的 `entry` 删除。

使用这种策略，每次申请一个 `timer` 时需要以 `O(1)` 的时间复杂度将一个申请 `append` 到链表尾部；触发时间到达时，需要以 `O(1)` 的时间复杂度将申请从头部删除，所以不会在频繁申请和频繁到期触发回调的过程中，因为这套策略而造成性能瓶颈。
