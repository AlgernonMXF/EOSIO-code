# memory
## memory.h

## memory.hpp
* **function**

```C++
//实现虚拟内存到内存的映射
void* sbrk(size_t num_bytes)

//动态分配内存
//从堆中获得指定字节的内存空间
//未初始化
void* malloc(size_t size)

//从堆中动态分配内存
//已初始化为0
//适合为数组申请空间，可以将size设置为数组元素的空间长度，count设置为数组元素数目
void* calloc(size_t count, size_t size)

//内存分配和内存释放
//将ptr指向的内存空间大小改为size
//若size小于ptr指向的空间大小，则不改变
//若size大于ptr指向的空间大小，将重新分配内存并将原有数据拷贝到新内存，同时释放之前的内存
void* realloc(void* ptr, size_t size)

//释放内存空间
void free(void* ptr)
```

