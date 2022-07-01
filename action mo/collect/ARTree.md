



[Adaptive Redix Tree](http://131.159.16.103/~leis/papers/ART.pdf) - 压缩的前缀树

拥有四种节点类型，每个节点包含2部分数据（key和叶子节点指针）

- Node4 = 长度为 4的数组存储 sorted的 key；接着为 长度为 4的数组存储叶子节点指针
- Node16  =  长度为 16的数组存储sorted的key；接着是对应的 叶子指针
- Node48 = 长度为 256的数组，按key直接index（因为Byte = 0~255），key就不排序了。而且存储的也不是key的值，而是“叶子指针”数组的index
- Node256 = 长度为 256的数组，存储的对应叶子节点的地址

注意，Node4和 Node16中 key数组是要根据 key进行排序的，而Node48 和Node256 是直接根据key 直接index的。

![image-20220609210753277](../../images/image-20220609210753277.png)