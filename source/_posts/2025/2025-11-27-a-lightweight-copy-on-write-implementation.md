---
title: 轻量级写时复制实现
date: 2025-11-27 21:29:55
updated: 2025-11-27 21:29:55
tags: [Java, Design Pattern]
---

Java 语言本身提供了两个 COW 数据结构 `CopyOnWriteArrayList` 和 `CopyOnWriteArraySet`，但在某些场景下，比如配置信息热更新、缓存系统数据刷新和一些读多写少且需要保证数据一致性的场景，它们太重了。以 `CopyOnWriteArrayList` 的 `add` 方法为例，它在实现时使用了 `synchronized` 同步代码块。在这些场景下，我们可以使用一种轻量级的实现方式，即**原子性地替换引用并利用 `volatile` 保证其他线程的可见性**。

<!-- more -->

```java
public class FooService {
    // 全局缓存，使用 volatile 确保引用更新的可见性
    private volatile List<String> cache = new ArrayList<>();

    public void load() {
        // 模拟从数据源加载新数据
        List<String> loaded = new ArrayList<>();
        loaded.add("a");
        loaded.add("b");
        // 原子性地替换缓存引用，利用 volatile 保证其他线程可见
        // 不直接修改原列表，避免并发读写问题
        this.cache = loaded;
    }

    public void usage() {
        // 创建当前缓存的本地快照，确保后续操作基于一致的数据视图
        // 即使其他线程调用 load() 更新缓存，也不会影响本次使用
        // 利用 volatile 读取最新值，无需加锁即可实现线程安全的读取
        List<String> backup = this.cache;
        // 后续操作应仅使用 backup，避免多次读取 this.cache 导致不一致
        for (String item : backup) {
            System.out.println(item);
        }
    }
}
```

这种实现方式可以称为*无锁读写模式*、*乐观读取模式*、*Safe Publication via Immutable Snapshots* 或者 *Copy-on-Write Reference*。下面是运行时引用变化示意图

{% asset_img reference.drawio.svg %}
