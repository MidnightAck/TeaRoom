# 内存管理

### 引用技术底层实现

通过isa指针和sidetables。isa指针在 [关于NSObject](bear://x-callback-url/open-note?id=FF0ADE02-3CFF-43C6-B905-68F03E3D0554-43776-0000F9685D53F19D)中有提起过，用来每一个实例指向其对应的类，以及每一个类指向他们的元类。isa除了存储这些类的地址信息后，在arm64架构之后，内部还多存了两个数据。一个`extra_rc`，是除当前实例外对该类的引用计数值，如果超过了19位，那么另一个`has_sidetable_rc`就会变为1。此时需要在一个叫sidetables的哈希表中查询引用计数的个数。 sidetables是多个sidetable的集合，每一个sidetable都是一个哈希表。这样的目的在于，如果只有一个哈希表，那么多线程进行查询的时候需要加锁保证线程安全，如果使用多个表，不同部分的sidetable进行查询的时候就可以并发。

