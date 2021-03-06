重点回顾
------------

- Redis 数据库中的每个键值对的键和值都是一个对象。

- Redis 共有字符串、列表、哈希、集合、有序集合五种类型的对象，
  每种类型的对象至少都有两种或以上的编码方式，
  不同的编码可以在不同的使用场景上优化对象的使用效率。

- 服务器在执行某些命令之前，
  会先检查给定键的类型能否执行指定的命令，
  而检查一个键的类型就是检查键的值对象的类型。

- Redis 的对象系统带有引用计数实现的内存回收机制，
  当一个对象不再被使用时，
  该对象所占用的内存就会被自动释放。

- Redis 会共享值为 ``0`` 到 ``9999`` 的字符串对象。

- 对象会记录自己的最后一次被访问的时间，
  这个时间可以用于计算对象的空转时间。
