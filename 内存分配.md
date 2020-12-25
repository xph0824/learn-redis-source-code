
源码文件

```
zmalloc.c
文件一般放的是变量、数组、函数的具体定义。
.c文件，以c为扩展名，一般存储具体功能的实现。

zmalloc.h
.h中一般放的是同名.c文件中定义的bai变量、数组、函数的声明，需要让.c外部使用的声明。
.h文件，称为头文件，一般存储类型的定义，函数的声明等。通常，头文件被.c文件包含，使用#include 语句。但值得注意的是，这只是一种约定，而非强制。
```

看一下zmalloc.h文件

```
/* Double expansion needed for stringification of macro values. */
#define __xstr(s) __str(s)
#define __str(s) #s


#if defined(USE_TCMALLOC)
#define ZMALLOC_LIB ("tcmalloc-" __xstr(TC_VERSION_MAJOR) "." __xstr(TC_VERSION_MINOR))
#include <google/tcmalloc.h>
#if (TC_VERSION_MAJOR == 1 && TC_VERSION_MINOR >= 6) || (TC_VERSION_MAJOR > 1)
#define HAVE_MALLOC_SIZE 1
#define zmalloc_size(p) tc_malloc_size(p)
#else
#error "Newer version of tcmalloc required"
#endif

可以看到对USE_TCMALLOC的宏定义，其中__xstr是宏值字符串化。如果定义了USE_TCMALLOC，也就是使用TCMALLOC，则定义ZMALLOC_LIB为tcmalloc-主版本号.次版本号，并且引入google/tcmalloc.h文件。如果版本是在次版本大于1.6或主版本号大于1的情况下，定义HAVE_MALLOC_SIZE为1，并定义zmalloc_size§为tc_malloc_size§。否则会输出需要更新的版本。
HAVE_MALLOC_SIZE宏定义是对zmalloc.c提供函数是否有zmalloc_size函数的控制。

#elif defined(USE_JEMALLOC)
#define ZMALLOC_LIB ("jemalloc-" __xstr(JEMALLOC_VERSION_MAJOR) "." __xstr(JEMALLOC_VERSION_MINOR) "." __xstr(JEMALLOC_VERSION_BUGFIX))
 #include <jemalloc/jemalloc.h>
 #if (JEMALLOC_VERSION_MAJOR == 2 && JEMALLOC_VERSION_MINOR >= 1) || (JEMALLOC_VERSION_MAJOR > 2)
#define HAVE_MALLOC_SIZE 1
#define zmalloc_size(p) je_malloc_usable_size(p)
#else
#error "Newer version of jemalloc required"
#endif
可以看到对USE_JEMALLOC的宏定义。如果定义了USE_JEMALLOC，也就是使用JEMALLOC，则定义JEMALLOC_LIB为jemalloc-主版本号.次版本号，并且引入jemalloc/jemalloc.h文件。如果版本是在次版本大于2.1或主版本号大于2的情况下，定义HAVE_MALLOC_SIZE为1，并定义zmalloc_size为je_malloc_usable_size。否则会输出需要更新 的版本。
 
#elif defined(__APPLE__)
#include <malloc/malloc.h>
#define HAVE_MALLOC_SIZE 1
#define zmalloc_size(p) malloc_size(p)
#endif

#ifndef ZMALLOC_LIB
#define ZMALLOC_LIB "libc"
#endif
如果jemalloc和tcmalloc都没有使用就用glibc的库函数
```




C语言中，内存分配的函数有malloc，free，relloc这3个函数，在Redis系统中对内存的分配又做了一次小封装。
先看一下在zmalloc.h头文件中define的一些API:

```
void *zmalloc(size_t size); /* 调用zmalloc申请size个大小的空间 */
void *zcalloc(size_t size); /* 调用系统函数calloc函数申请空间 */
void *zrealloc(void *ptr, size_t size); /* 原内存重新调整空间为size的大小 */
void zfree(void *ptr); /* 释放空间方法，并更新used_memory的值 */
char *zstrdup(const char *s); /* 字符串复制方法 */
size_t zmalloc_used_memory(void); /* 获取当前已经占用的内存大小 */
void zmalloc_enable_thread_safeness(void); /* 是否设置线程安全模式 */
void zmalloc_set_oom_handler(void (*oom_handler)(size_t)); /* 可自定义设置内存溢出的处理方法 */
float zmalloc_get_fragmentation_ratio(size_t rss); /* 所给大小与已使用内存大小之比 */
size_t zmalloc_get_rss(void);
size_t zmalloc_get_private_dirty(void); /* 获取私有的脏数据大小 */
void zlibc_free(void *ptr);  /* 原始系统free释放方法 */
```

介绍几个概念和变量:

```
static size_t used_memory = 0;
static int zmalloc_thread_safe = 0;
pthread_mutex_t used_memory_mutex = PTHREAD_MUTEX_INITIALIZER;
```
第一个used_memory，这是系统已经使用了多少的内存大小，在全局只维护了这么一个变量的大小，说明作者希望根据此来分析出当前的内存的使用情况;
第二个zmalloc_thread_safe，这指的是线程安全模式状态，下面的mutext，就是为此服务端，这个在操作系统中就出现过。据此，我们大概知道了，Redis在代码的malloc等操作的时候，会根据创建的大小，会更新 used_memory，并操作的模式会有线程安全和不安去模式的区分。


看一下zmalloc.c文件

```
/* 在对内存空间做使用的时候，进行了加锁控制 */
#define update_zmalloc_stat_add(__n) do { \
    pthread_mutex_lock(&used_memory_mutex); \
    used_memory += (__n); \
    pthread_mutex_unlock(&used_memory_mutex); \
} while(0)
```

上面的函数操作就是线程安全模式时候的一个操作，通过锁操作实现对于used_memory的控制，是对这个变量做控制，避免了这个数值出现脏数据的可能。
zmalloc针对used_memory的多线程保护的处理：used_memory是使用的内存量，而used_memory_mutex是针对used_memory变量的保护锁。zmalloc_thread_safe是对zmalloc线程安全性的开关。

当使用原子操作时，就使用__sync_add_and_fetch和__sync_sub_and_fetch。否则就使用线程锁，update_zmalloc_stat_add和update_zmalloc_stat_sub就是提供的对加锁解锁操作的封装。
```

```


```
/* 调用zmalloc申请size个大小的空间 */
void *zmalloc(size_t size) {
	//实际调用的还是malloc函数
    void *ptr = malloc(size+PREFIX_SIZE);
	
	//如果申请的结果为null，说明发生了oom,调用oom的处理方法
    if (!ptr) zmalloc_oom_handler(size);
#ifdef HAVE_MALLOC_SIZE
	//更新used_memory的大小
    update_zmalloc_stat_alloc(zmalloc_size(ptr));
    return ptr;
#else
    *((size_t*)ptr) = size;
    update_zmalloc_stat_alloc(size+PREFIX_SIZE);
    return (char*)ptr+PREFIX_SIZE;
#endif
}

```

zmalloc的具体实现，调用的还是malloc的C语言方法，做了OOM的异常处理，然后更新大小。在update_zmalloc_stat_alloc方法里面:

```

/* 申请新的_n大小的内存，分为线程安全，和线程不安全的模式 */
#define update_zmalloc_stat_alloc(__n) do { \
    size_t _n = (__n); \
    if (_n&(sizeof(long)-1)) _n += sizeof(long)-(_n&(sizeof(long)-1)); \
    if (zmalloc_thread_safe) { \
        update_zmalloc_stat_add(_n); \
    } else { \
        used_memory += _n; \
    } \
} while(0)
```

同理zfree()的操作就是上面的反操作，调用free方法，把used_memory的值，做减少操作。在APIL里面还出现了zcalloc方法，下面函数代码中我分析一下这个函数和malloc到底有什么不同:

```

/* 调用系统函数calloc函数申请空间 */
void *zcalloc(size_t size) {
	//calloc与malloc的意思一样，不过参数不一样
	//void *calloc(size_t numElements,size_t sizeOfElement),;numElements * sizeOfElement才是最终的内存的大小
	//所在这里就是申请一块大小为size+PREFIX_SIZE的空间
    void *ptr = calloc(1, size+PREFIX_SIZE);
 
    if (!ptr) zmalloc_oom_handler(size);
#ifdef HAVE_MALLOC_SIZE
    update_zmalloc_stat_alloc(zmalloc_size(ptr));
    return ptr;
#else
    *((size_t*)ptr) = size;
    update_zmalloc_stat_alloc(size+PREFIX_SIZE);
    return (char*)ptr+PREFIX_SIZE;
#endif
}
```

自定义一个更合理的内存溢出的处理函数，更满足系统的需要:
```
/* 可自定义设置内存溢出的处理方法 */
void zmalloc_set_oom_handler(void (*oom_handler)(size_t)) {
    zmalloc_oom_handler = oom_handler;
}
```




