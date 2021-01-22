
dict，即字典，也被称为哈希表hashtable。在redis的五大数据结构中，有如下两种情形会使用dict结构：
  1）hash：数据量小的时候使用ziplist，量大时使用dict
  2）zset：数据量小的时候使用ziplist，数据量大的时候使用skiplist + dict
  
#### dict.h 文件

```

/* Hash Tables Implementation.
*
* This file implements in-memory hash tables with insert/del/replace/find/
* get-random-element operations. Hash tables will auto-resize if needed
* tables of power of two in size are used, collisions are handled by
* chaining. See the source code for more information... :)
*
* Copyright (c) 2006-2012, Salvatore Sanfilippo <antirez at gmail dot com>
* All rights reserved.
*
* Redistribution and use in source and binary forms, with or without
* modification, are permitted provided that the following conditions are met:
*
*   * Redistributions of source code must retain the above copyright notice,
*     this list of conditions and the following disclaimer.
*   * Redistributions in binary form must reproduce the above copyright
*     notice, this list of conditions and the following disclaimer in the
*     documentation and/or other materials provided with the distribution.
*   * Neither the name of Redis nor the names of its contributors may be used
*     to endorse or promote products derived from this software without
*     specific prior written permission.
*
* THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
* AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
* IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
* ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS BE
* LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
* CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
* SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
* INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
* CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
* ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
* POSSIBILITY OF SUCH DAMAGE.
*/  
  
#include <stdint.h>  
  
#ifndef __DICT_H  
#define __DICT_H  
  
/* 定义了成功与错误的值 */  
#define DICT_OK 0  
#define DICT_ERR 1  
  
/* Unused arguments generate annoying warnings... */  
/* dict没有用到时，用来提示警告的 */  
#define DICT_NOTUSED(V) ((void) V)  
  
/* 字典结构体，保存K-V值的结构体 */  
typedef struct dictEntry {  
    //字典key函数指针  
    void *key;  
    union {  
        void *val;  
        //无符号整型值  
        uint64_t u64;  
        //有符号整型值  
        int64_t s64;  
        double d;  
    } v;  
    //下一字典结点  
    struct dictEntry *next;  
} dictEntry;  
  
/* 字典类型 */  
typedef struct dictType {  
    //哈希计算方法，返回整形变量  
    unsigned int (*hashFunction)(const void *key);  
    //复制key方法  
    void *(*keyDup)(void *privdata, const void *key);  
    //复制val方法  
    void *(*valDup)(void *privdata, const void *obj);  
    //key值比较方法  
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);  
    //key的析构函数  
    void (*keyDestructor)(void *privdata, void *key);  
    //val的析构函数  
    void (*valDestructor)(void *privdata, void *obj);  
} dictType;  
  
/* This is our hash table structure. Every dictionary has two of this as we
* implement incremental rehashing, for the old to the new table. */  
/* 哈希表结构体 */  
typedef struct dictht {  
    //字典实体  
    dictEntry **table;  
    //表格可容纳字典数量  
    unsigned long size;  
    unsigned long sizemask;  
    //正在被使用的数量  
    unsigned long used;  
} dictht;  
  
/* 字典主操作类 */  
typedef struct dict {  
    //字典类型  
    dictType *type;  
    //私有数据指针  
    void *privdata;  
    //字典哈希表，共2张，一张旧的，一张新的  
    dictht ht[2];  
    //重定位哈希时的下标  
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */  
    //当前迭代器数量  
    int iterators; /* number of iterators currently running */  
} dict;  
  
/* If safe is set to 1 this is a safe iterator, that means, you can call
* dictAdd, dictFind, and other functions against the dictionary even while
* iterating. Otherwise it is a non safe iterator, and only dictNext()
* should be called while iterating. */  
/* 字典迭代器，如果是安全迭代器，这safe设置为1，可以调用dicAdd，dictFind */  
/* 如果是不安全的，则只能调用dicNext方法*/  
typedef struct dictIterator {  
    //当前字典  
    dict *d;  
    //下标  
    long index;  
    //表格，和安全值的表格代表的是旧的表格，还是新的表格  
    int table, safe;  
    //字典实体  
    dictEntry *entry, *nextEntry;  
    /* unsafe iterator fingerprint for misuse detection. */  
    /* 指纹标记，避免不安全的迭代器滥用现象 */  
    long long fingerprint;  
} dictIterator;  
  
/* 字典扫描方法 */  
typedef void (dictScanFunction)(void *privdata, const dictEntry *de);  
  
/* This is the initial size of every hash table */  
/* 初始化哈希表的数目 */  
#define DICT_HT_INITIAL_SIZE     4  
  
/* ------------------------------- Macros ------------------------------------*/  
/* 字典释放val函数时候调用，如果dict中的dictType定义了这个函数指针， */  
#define dictFreeVal(d, entry) \  
    if ((d)->type->valDestructor) \  
        (d)->type->valDestructor((d)->privdata, (entry)->v.val)  
      
/* 字典val函数复制时候调用，如果dict中的dictType定义了这个函数指针， */  
#define dictSetVal(d, entry, _val_) do { \  
    if ((d)->type->valDup) \  
        entry->v.val = (d)->type->valDup((d)->privdata, _val_); \  
    else \  
        entry->v.val = (_val_); \  
} while(0)  
  
/* 设置dictEntry中共用体v中有符号类型的值 */  
#define dictSetSignedIntegerVal(entry, _val_) \  
    do { entry->v.s64 = _val_; } while(0)  
  
/* 设置dictEntry中共用体v中无符号类型的值 */  
#define dictSetUnsignedIntegerVal(entry, _val_) \  
    do { entry->v.u64 = _val_; } while(0)  
  
/* 设置dictEntry中共用体v中double类型的值 */  
#define dictSetDoubleVal(entry, _val_) \  
    do { entry->v.d = _val_; } while(0)  
  
/* 调用dictType定义的key析构函数 */  
#define dictFreeKey(d, entry) \  
    if ((d)->type->keyDestructor) \  
        (d)->type->keyDestructor((d)->privdata, (entry)->key)  
  
/* 调用dictType定义的key复制函数，没有定义直接赋值 */  
#define dictSetKey(d, entry, _key_) do { \  
    if ((d)->type->keyDup) \  
        entry->key = (d)->type->keyDup((d)->privdata, _key_); \  
    else \  
        entry->key = (_key_); \  
} while(0)  
  
/* 调用dictType定义的key比较函数，没有定义直接key值直接比较 */  
#define dictCompareKeys(d, key1, key2) \  
    (((d)->type->keyCompare) ? \  
        (d)->type->keyCompare((d)->privdata, key1, key2) : \  
        (key1) == (key2))  
  
#define dictHashKey(d, key) (d)->type->hashFunction(key)   //哈希定位方法  
#define dictGetKey(he) ((he)->key)    //获取dictEntry的key值  
#define dictGetVal(he) ((he)->v.val)  //获取dicEntry中共用体v中定义的val值  
#define dictGetSignedIntegerVal(he) ((he)->v.s64) //获取dicEntry中共用体v中定义的有符号值  
#define dictGetUnsignedIntegerVal(he) ((he)->v.u64)  //获取dicEntry中共用体v中定义的无符号值  
#define dictGetDoubleVal(he) ((he)->v.d)  //获取dicEntry中共用体v中定义的double类型值  
#define dictSlots(d) ((d)->ht[0].size+(d)->ht[1].size)  //获取dict字典中总的表大小  
#define dictSize(d) ((d)->ht[0].used+(d)->ht[1].used)   //获取dict字典中总的表的总正在被使用的数量  
#define dictIsRehashing(d) ((d)->rehashidx != -1)   //字典有无被重定位过  
  
/* API */  
dict *dictCreate(dictType *type, void *privDataPtr);   //创建dict字典总类  
int dictExpand(dict *d, unsigned long size);    //字典扩增方法  
int dictAdd(dict *d, void *key, void *val);    //字典根据key, val添加一个字典集  
dictEntry *dictAddRaw(dict *d, void *key);     //字典添加一个只有key值的dicEntry  
int dictReplace(dict *d, void *key, void *val); //替代dict中一个字典集  
dictEntry *dictReplaceRaw(dict *d, void *key);  //替代dict中的一个字典，只提供一个key值  
int dictDelete(dict *d, const void *key);    //根据key删除一个字典集  
int dictDeleteNoFree(dict *d, const void *key);  //字典集删除无、不调用free方法  
void dictRelease(dict *d);   //释放整个dict  
dictEntry * dictFind(dict *d, const void *key);  //根据key寻找字典集  
void *dictFetchValue(dict *d, const void *key);  //根据key值寻找相应的val值  
int dictResize(dict *d);  //重新计算大小  
dictIterator *dictGetIterator(dict *d); //获取字典迭代器  
dictIterator *dictGetSafeIterator(dict *d);  //获取字典安全迭代器   
dictEntry *dictNext(dictIterator *iter);   //根据字典迭代器获取字典集的下一字典集  
void dictReleaseIterator(dictIterator *iter); //释放迭代器  
dictEntry *dictGetRandomKey(dict *d);  //随机获取一个字典集  
void dictPrintStats(dict *d);  //打印当前字典状态  
unsigned int dictGenHashFunction(const void *key, int len); //输入的key值，目标长度，此方法帮你计算出索引值  
unsigned int dictGenCaseHashFunction(const unsigned char *buf, int len); //这里提供了一种比较简单的哈希算法  
void dictEmpty(dict *d, void(callback)(void*)); //清空字典  
void dictEnableResize(void);  //启用调整方法  
void dictDisableResize(void); //禁用调整方法  
int dictRehash(dict *d, int n); //hash重定位，主要从旧的表映射到新表中,分n轮定位  
int dictRehashMilliseconds(dict *d, int ms);  //在给定时间内，循环执行哈希重定位  
void dictSetHashFunctionSeed(unsigned int initval); //设置哈希方法种子  
unsigned int dictGetHashFunctionSeed(void);  //获取哈希种子  
unsigned long dictScan(dict *d, unsigned long v, dictScanFunction *fn, void *privdata); //字典扫描方法  
  
/* Hash table types */  
/* 哈希表类型  */  
extern dictType dictTypeHeapStringCopyKey;  
extern dictType dictTypeHeapStrings;  
extern dictType dictTypeHeapStringCopyKeyValue;  
  
#endif /* __DICT_H */  

```

#### dict.c

```
/* Hash Tables Implementation.
*
* This file implements in memory hash tables with insert/del/replace/find/
* get-random-element operations. Hash tables will auto resize if needed
* tables of power of two in size are used, collisions are handled by
* chaining. See the source code for more information... :)
*
* Copyright (c) 2006-2012, Salvatore Sanfilippo <antirez at gmail dot com>
* All rights reserved.
*
* Redistribution and use in source and binary forms, with or without
* modification, are permitted provided that the following conditions are met:
*
*   * Redistributions of source code must retain the above copyright notice,
*     this list of conditions and the following disclaimer.
*   * Redistributions in binary form must reproduce the above copyright
*     notice, this list of conditions and the following disclaimer in the
*     documentation and/or other materials provided with the distribution.
*   * Neither the name of Redis nor the names of its contributors may be used
*     to endorse or promote products derived from this software without
*     specific prior written permission.
*
* THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
* AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
* IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
* ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS BE
* LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
* CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
* SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
* INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
* CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
* ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
* POSSIBILITY OF SUCH DAMAGE.
*/  
  
#include "fmacros.h"  
  
#include <stdio.h>  
#include <stdlib.h>  
#include <string.h>  
#include <stdarg.h>  
#include <limits.h>  
#include <sys/time.h>  
#include <ctype.h>  
  
#include "dict.h"  
#include "zmalloc.h"  
#include "redisassert.h"  
  
/* Using dictEnableResize() / dictDisableResize() we make possible to
* enable/disable resizing of the hash table as needed. This is very important
* for Redis, as we use copy-on-write and don't want to move too much memory
* around when there is a child performing saving operations.
*
* Note that even when dict_can_resize is set to 0, not all resizes are
* prevented: a hash table is still allowed to grow if the ratio between
* the number of elements and the buckets > dict_force_resize_ratio. */  
/* redis用了dictEnableResize() / dictDisableResize()方法可以重新调整哈希表的长度，
*因为redis采用的是写时复制的算法，不会挪动太多的内存，只有当调整数量大于一定比例才可能有效 */  
static int dict_can_resize = 1;  
static unsigned int dict_force_resize_ratio = 5;  
  
/* -------------------------- private prototypes ---------------------------- */  
/* 私有方法 */  
static int _dictExpandIfNeeded(dict *ht);    //字典是否需要扩展  
static unsigned long _dictNextPower(unsigned long size);  
static int _dictKeyIndex(dict *ht, const void *key);  
static int _dictInit(dict *ht, dictType *type, void *privDataPtr);  //字典初始化方法  
  
/* -------------------------- hash functions -------------------------------- */  
/* 哈希索引计算的方法 */  
  
/* Thomas Wang's 32 bit Mix Function */  
/* Thomas Wang's 32 bit Mix 的哈希算法直接输入key值，获取索引值，据说这种冲突的概率很低 */  
unsigned int dictIntHashFunction(unsigned int key)  
{  
    key += ~(key << 15);  
    key ^=  (key >> 10);  
    key +=  (key << 3);  
    key ^=  (key >> 6);  
    key += ~(key << 11);  
    key ^=  (key >> 16);  
    return key;  
}  
  
//哈希方法种子，跟产生随机数的种子作用应该是一样的  
static uint32_t dict_hash_function_seed = 5381;  
  
/* 重设哈希种子 */  
void dictSetHashFunctionSeed(uint32_t seed) {  
    dict_hash_function_seed = seed;  
}  
  
/* 获取哈希种子 */  
uint32_t dictGetHashFunctionSeed(void) {  
    return dict_hash_function_seed;  
}  
  
/* MurmurHash2, by Austin Appleby
* Note - This code makes a few assumptions about how your machine behaves -
* 1. We can read a 4-byte value from any address without crashing
* 2. sizeof(int) == 4
*
* And it has a few limitations -
*
* 1. It will not work incrementally.
* 2. It will not produce the same results on little-endian and big-endian
*    machines.
*/  
/* 输入的key值，目标长度，此方法帮你计算出索引值，此方法特别表明，
*  不会因为机器之间高低位存储的不同而产生相同的结果 */  
unsigned int dictGenHashFunction(const void *key, int len) {  
    /* 'm' and 'r' are mixing constants generated offline.
     They're not really 'magic', they just happen to work well.  */  
    //seed种子，m，r的值都将会参与到计算中  
    uint32_t seed = dict_hash_function_seed;  
    const uint32_t m = 0x5bd1e995;  
    const int r = 24;  
  
    /* Initialize the hash to a 'random' value */  
    uint32_t h = seed ^ len;  
  
    /* Mix 4 bytes at a time into the hash */  
    const unsigned char *data = (const unsigned char *)key;  
  
    while(len >= 4) {  
        uint32_t k = *(uint32_t*)data;  
  
        k *= m;  
        k ^= k >> r;  
        k *= m;  
  
        h *= m;  
        h ^= k;  
  
        data += 4;  
        len -= 4;  
    }  
  
    /* Handle the last few bytes of the input array  */  
    switch(len) {  
    case 3: h ^= data[2] << 16;  
    case 2: h ^= data[1] << 8;  
    case 1: h ^= data[0]; h *= m;  
    };  
  
    /* Do a few final mixes of the hash to ensure the last few
     * bytes are well-incorporated. */  
    h ^= h >> 13;  
    h *= m;  
    h ^= h >> 15;  
  
    return (unsigned int)h;  
}  
  
/* And a case insensitive hash function (based on djb hash) */  
/* 这里提供了一种比较简单的哈希算法 */  
unsigned int dictGenCaseHashFunction(const unsigned char *buf, int len) {  
    //以djb hash为基础，俗称“times33”就是不断的乘33  
    //几乎所有的流行的hash map都采用了DJB hash function  
    unsigned int hash = (unsigned int)dict_hash_function_seed;  
  
    while (len--)  
        hash = ((hash << 5) + hash) + (tolower(*buf++)); /* hash * 33 + c */  
    return hash;  
}  
  
/* ----------------------------- API implementation ------------------------- */  
  
/* Reset a hash table already initialized with ht_init().
* NOTE: This function should only be called by ht_destroy(). */  
/* 重置哈希表方法，只在ht_destroy时使用 */  
static void _dictReset(dictht *ht)  
{  
    //清空相应的变量，ht->table的类型其实是dictEntry，叫table名字太有歧义了  
    ht->table = NULL;  
    ht->size = 0;  
    ht->sizemask = 0;  
    ht->used = 0;  
}  
  
/* Create a new hash table */  
/* 创建dict操作类 */  
dict *dictCreate(dictType *type,  
        void *privDataPtr)  
{  
    dict *d = zmalloc(sizeof(*d));  
      
    //创建好空间之后调用初始化方法  
    _dictInit(d,type,privDataPtr);  
    return d;  
}  
  
/* Initialize the hash table */  
/* 初始化dict类中的type，ht等变量 */  
int _dictInit(dict *d, dictType *type,  
        void *privDataPtr)  
{  
    //重置2个ht哈希表  
    _dictReset(&d->ht[0]);  
    _dictReset(&d->ht[1]);  
    //赋值dictType  
    d->type = type;  
    d->privdata = privDataPtr;  
    //-1代表还没有rehash过，  
    d->rehashidx = -1;  
    //当前使用中的迭代器为0  
    d->iterators = 0;  
      
    //返回DICT_OK，代表初始化成功  
    return DICT_OK;  
}  
  
/* Resize the table to the minimal size that contains all the elements,
* but with the invariant of a USED/BUCKETS ratio near to <= 1 */  
/* 调整哈希表，用最少的值容纳所有的字典集合 */  
int dictResize(dict *d)  
{  
    int minimal;  
  
    //如果系统默认调整值不大于0或已经调rehash过的就提示出错，拒绝操作  
    if (!dict_can_resize || dictIsRehashing(d)) return DICT_ERR;  
      
    //最少数等于哈希标准鸿正在使用的数  
    minimal = d->ht[0].used;  
    if (minimal < DICT_HT_INITIAL_SIZE)  
        minimal = DICT_HT_INITIAL_SIZE;  
      
    //调用expand扩容  
    return dictExpand(d, minimal);  
}  
  
/* Expand or create the hash table */  
/* 哈希表扩增方法 */  
int dictExpand(dict *d, unsigned long size)  
{  
    dictht n; /* the new hash table */  
    //获取调整值，以2的幂次向上取  
    unsigned long realsize = _dictNextPower(size);  
  
    /* the size is invalid if it is smaller than the number of
     * elements already inside the hash table */  
     //再次判断数量符合不符合  
    if (dictIsRehashing(d) || d->ht[0].used > size)  
        return DICT_ERR;  
  
    /* Allocate the new hash table and initialize all pointers to NULL */  
    //初始化大小  
    n.size = realsize;  
    n.sizemask = realsize-1;  
    //为表格申请realsize个字典集的大小  
    n.table = zcalloc(realsize*sizeof(dictEntry*));  
    n.used = 0;  
  
    /* Is this the first initialization? If so it's not really a rehashing
     * we just set the first hash table so that it can accept keys. */  
    if (d->ht[0].table == NULL) {  
        d->ht[0] = n;  
        return DICT_OK;  
    }  
  
    /* Prepare a second hash table for incremental rehashing */  
    //赋值给第二张表格  
    d->ht[1] = n;  
    d->rehashidx = 0;  
    return DICT_OK;  
}  
  
/* Performs N steps of incremental rehashing. Returns 1 if there are still
* keys to move from the old to the new hash table, otherwise 0 is returned.
* Note that a rehashing step consists in moving a bucket (that may have more
* than one key as we use chaining) from the old to the new hash table. */  
/* hash重定位，主要从旧的表映射到新表中
* 如果返回1说明旧的表中还存在key迁移到新表中，0代表没有 */  
int dictRehash(dict *d, int n) {  
    if (!dictIsRehashing(d)) return 0;  
      
    /* 根据参数分n步多次循环操作 */  
    while(n--) {  
        dictEntry *de, *nextde;  
  
        /* Check if we already rehashed the whole table... */  
        if (d->ht[0].used == 0) {  
            zfree(d->ht[0].table);  
            d->ht[0] = d->ht[1];  
            _dictReset(&d->ht[1]);  
            d->rehashidx = -1;  
            return 0;  
        }  
  
        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */  
        assert(d->ht[0].size > (unsigned long)d->rehashidx);  
        while(d->ht[0].table[d->rehashidx] == NULL) d->rehashidx++;  
        de = d->ht[0].table[d->rehashidx];  
        /* Move all the keys in this bucket from the old to the new hash HT */  
        /* 移动的关键操作 */  
        while(de) {  
            unsigned int h;  
  
            nextde = de->next;  
            /* Get the index in the new hash table */  
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;  
            de->next = d->ht[1].table[h];  
            d->ht[1].table[h] = de;  
            d->ht[0].used--;  
            d->ht[1].used++;  
            de = nextde;  
        }  
        d->ht[0].table[d->rehashidx] = NULL;  
        d->rehashidx++;  
    }  
    return 1;  
}  
  
/* 获取当前毫秒的时间 */  
long long timeInMilliseconds(void) {  
    struct timeval tv;  
  
    gettimeofday(&tv,NULL);  
    return (((long long)tv.tv_sec)*1000)+(tv.tv_usec/1000);  
}  
  
/* Rehash for an amount of time between ms milliseconds and ms+1 milliseconds */  
/* 在给定时间内，循环执行哈希重定位 */  
int dictRehashMilliseconds(dict *d, int ms) {  
    long long start = timeInMilliseconds();  
    int rehashes = 0;  
  
    while(dictRehash(d,100)) {  
        //重定位的次数累加  
        rehashes += 100;  
        //时间超出给定时间范围，则终止  
        if (timeInMilliseconds()-start > ms) break;  
    }  
    return rehashes;  
}  
  
/* This function performs just a step of rehashing, and only if there are
* no safe iterators bound to our hash table. When we have iterators in the
* middle of a rehashing we can't mess with the two hash tables otherwise
* some element can be missed or duplicated.
*
* This function is called by common lookup or update operations in the
* dictionary so that the hash table automatically migrates from H1 to H2
* while it is actively used. */  
/* 当没有迭代器时候，进行重定位算法 */  
static void _dictRehashStep(dict *d) {  
    if (d->iterators == 0) dictRehash(d,1);  
}  
  
/* Add an element to the target hash table */  
/* 添加一个dicEntry */  
int dictAdd(dict *d, void *key, void *val)  
{  
    dictEntry *entry = dictAddRaw(d,key);  
  
    if (!entry) return DICT_ERR;  
    dictSetVal(d, entry, val);  
    return DICT_OK;  
}  
  
/* Low level add. This function adds the entry but instead of setting
* a value returns the dictEntry structure to the user, that will make
* sure to fill the value field as he wishes.
*
* This function is also directly exposed to user API to be called
* mainly in order to store non-pointers inside the hash value, example:
*
* entry = dictAddRaw(dict,mykey);
* if (entry != NULL) dictSetSignedIntegerVal(entry,1000);
*
* Return values:
*
* If key already exists NULL is returned.
* If key was added, the hash entry is returned to be manipulated by the caller.
*/  
/* 添加一个指定key值的Entry */  
dictEntry *dictAddRaw(dict *d, void *key)  
{  
    int index;  
    dictEntry *entry;  
    dictht *ht;  
  
    if (dictIsRehashing(d)) _dictRehashStep(d);  
  
    /* Get the index of the new element, or -1 if
     * the element already exists. */  
    /* 如果指定的key已经存在，则直接返回NULL说明添加失败 */  
    if ((index = _dictKeyIndex(d, key)) == -1)  
        return NULL;  
  
    /* Allocate the memory and store the new entry */  
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];  
    entry = zmalloc(sizeof(*entry));  
    entry->next = ht->table[index];  
    ht->table[index] = entry;  
    ht->used++;  
  
    /* Set the hash entry fields. */  
    dictSetKey(d, entry, key);  
    return entry;  
}  
  
/* Add an element, discarding the old if the key already exists.
* Return 1 if the key was added from scratch, 0 if there was already an
* element with such key and dictReplace() just performed a value update
* operation. */  
/* 替换一个子字典集，如果不存在直接添加，存在，覆盖val的值 */  
int dictReplace(dict *d, void *key, void *val)  
{  
    dictEntry *entry, auxentry;  
  
    /* Try to add the element. If the key
     * does not exists dictAdd will suceed. */  
    //不存在，这个key直接添加  
    if (dictAdd(d, key, val) == DICT_OK)  
        return 1;  
    /* It already exists, get the entry */  
    entry = dictFind(d, key);  
    /* Set the new value and free the old one. Note that it is important
     * to do that in this order, as the value may just be exactly the same
     * as the previous one. In this context, think to reference counting,
     * you want to increment (set), and then decrement (free), and not the
     * reverse. */  
    //赋值方法  
    auxentry = *entry;  
    dictSetVal(d, entry, val);  
    dictFreeVal(d, &auxentry);  
    return 0;  
}  
  
/* dictReplaceRaw() is simply a version of dictAddRaw() that always
* returns the hash entry of the specified key, even if the key already
* exists and can't be added (in that case the entry of the already
* existing key is returned.)
*
* See dictAddRaw() for more information. */  
/* 添加字典，没有函数方法，如果存在，就不添加 */  
dictEntry *dictReplaceRaw(dict *d, void *key) {  
    dictEntry *entry = dictFind(d,key);  
  
    return entry ? entry : dictAddRaw(d,key);  
}  
  
/* Search and remove an element */  
/* 删除给定key的结点，可控制是否调用释放方法 */  
static int dictGenericDelete(dict *d, const void *key, int nofree)  
{  
    unsigned int h, idx;  
    dictEntry *he, *prevHe;  
    int table;  
  
    if (d->ht[0].size == 0) return DICT_ERR; /* d->ht[0].table is NULL */  
    if (dictIsRehashing(d)) _dictRehashStep(d);  
    //计算key对应的哈希索引  
    h = dictHashKey(d, key);  
  
    for (table = 0; table <= 1; table++) {  
        idx = h & d->ht[table].sizemask;  
        //找到具体的索引对应的结点  
        he = d->ht[table].table[idx];  
        prevHe = NULL;  
        while(he) {  
            if (dictCompareKeys(d, key, he->key)) {  
                /* Unlink the element from the list */  
                if (prevHe)  
                    prevHe->next = he->next;  
                else  
                    d->ht[table].table[idx] = he->next;  
                if (!nofree) {  
                    //判断是否需要调用dict定义的free方法  
                    dictFreeKey(d, he);  
                    dictFreeVal(d, he);  
                }  
                zfree(he);  
                d->ht[table].used--;  
                return DICT_OK;  
            }  
            prevHe = he;  
            he = he->next;  
        }  
        if (!dictIsRehashing(d)) break;  
    }  
    return DICT_ERR; /* not found */  
}  
  
/* 会调用free方法的删除方法 */  
int dictDelete(dict *ht, const void *key) {  
    return dictGenericDelete(ht,key,0);  
}  
  
/* 不会调用free方法的删除方法 */  
int dictDeleteNoFree(dict *ht, const void *key) {  
    return dictGenericDelete(ht,key,1);  
}  
  
/* Destroy an entire dictionary */  
/* 清空整个哈希表 */  
int _dictClear(dict *d, dictht *ht, void(callback)(void *)) {  
    unsigned long i;  
  
    /* Free all the elements */  
    for (i = 0; i < ht->size && ht->used > 0; i++) {  
        dictEntry *he, *nextHe;  
  
        //每次情况会调用回调方法  
        if (callback && (i & 65535) == 0) callback(d->privdata);  
  
        if ((he = ht->table[i]) == NULL) continue;  
        while(he) {  
            //依次释放结点  
            nextHe = he->next;  
            dictFreeKey(d, he);  
            dictFreeVal(d, he);  
            zfree(he);  
            ht->used--;  
            he = nextHe;  
        }  
    }  
    /* Free the table and the allocated cache structure */  
    zfree(ht->table);  
    /* Re-initialize the table */  
    _dictReset(ht);  
    return DICT_OK; /* never fails */  
}  
  
/* Clear & Release the hash table */  
/* 重置字典总类，清空2张表 */  
void dictRelease(dict *d)  
{  
    _dictClear(d,&d->ht[0],NULL);  
    _dictClear(d,&d->ht[1],NULL);  
    zfree(d);  
}  
  
/* 根据key返回具体的字典集 */  
dictEntry *dictFind(dict *d, const void *key)  
{  
    dictEntry *he;  
    unsigned int h, idx, table;  
  
    if (d->ht[0].size == 0) return NULL; /* We don't have a table at all */  
    if (dictIsRehashing(d)) _dictRehashStep(d);  
    h = dictHashKey(d, key);  
    for (table = 0; table <= 1; table++) {  
        idx = h & d->ht[table].sizemask;  
        he = d->ht[table].table[idx];  
        while(he) {  
            if (dictCompareKeys(d, key, he->key))  
                return he;  
            he = he->next;  
        }  
        if (!dictIsRehashing(d)) return NULL;  
    }  
    return NULL;  
}  
  
/* 获取目标字典集的方法 */  
void *dictFetchValue(dict *d, const void *key) {  
    dictEntry *he;  
  
    he = dictFind(d,key);  
    /* 获取字典集的方法 */  
    return he ? dictGetVal(he) : NULL;  
}  
  
/* A fingerprint is a 64 bit number that represents the state of the dictionary
* at a given time, it's just a few dict properties xored together.
* When an unsafe iterator is initialized, we get the dict fingerprint, and check
* the fingerprint again when the iterator is released.
* If the two fingerprints are different it means that the user of the iterator
* performed forbidden operations against the dictionary while iterating. */  
/* 通过指纹来禁止每个不安全的哈希迭代器的非法操作，每个不安全迭代器只能有一个指纹 */  
long long dictFingerprint(dict *d) {  
    long long integers[6], hash = 0;  
    int j;  
  
    integers[0] = (long) d->ht[0].table;  
    integers[1] = d->ht[0].size;  
    integers[2] = d->ht[0].used;  
    integers[3] = (long) d->ht[1].table;  
    integers[4] = d->ht[1].size;  
    integers[5] = d->ht[1].used;  
  
    /* We hash N integers by summing every successive integer with the integer
     * hashing of the previous sum. Basically:
     *
     * Result = hash(hash(hash(int1)+int2)+int3) ...
     *
     * This way the same set of integers in a different order will (likely) hash
     * to a different number. */  
    for (j = 0; j < 6; j++) {  
        hash += integers[j];  
        /* For the hashing step we use Tomas Wang's 64 bit integer hash. */  
        hash = (~hash) + (hash << 21); // hash = (hash << 21) - hash - 1;  
        hash = hash ^ (hash >> 24);  
        hash = (hash + (hash << 3)) + (hash << 8); // hash * 265  
        hash = hash ^ (hash >> 14);  
        hash = (hash + (hash << 2)) + (hash << 4); // hash * 21  
        hash = hash ^ (hash >> 28);  
        hash = hash + (hash << 31);  
    }  
    return hash;  
}  
  
/* 获取哈希迭代器，默认不安全的 */  
dictIterator *dictGetIterator(dict *d)  
{  
    dictIterator *iter = zmalloc(sizeof(*iter));  
  
    iter->d = d;  
    iter->table = 0;  
    iter->index = -1;  
    iter->safe = 0;  
    iter->entry = NULL;  
    iter->nextEntry = NULL;  
    return iter;  
}  
  
/* 获取安全哈希迭代器 */  
dictIterator *dictGetSafeIterator(dict *d) {  
    dictIterator *i = dictGetIterator(d);  
  
    i->safe = 1;  
    return i;  
}  
  
/* 迭代器获取下一个集合点 */  
dictEntry *dictNext(dictIterator *iter)  
{  
    while (1) {  
        if (iter->entry == NULL) {  
            dictht *ht = &iter->d->ht[iter->table];  
            if (iter->index == -1 && iter->table == 0) {  
                //如果迭代器index下标为-1说明还没开始使用，设置迭代器的指纹或增加引用计数量  
                if (iter->safe)  
                    iter->d->iterators++;  
                else  
                    iter->fingerprint = dictFingerprint(iter->d);  
            }  
            //迭代器下标递增  
            iter->index++;  
            if (iter->index >= (long) ht->size) {  
                if (dictIsRehashing(iter->d) && iter->table == 0) {  
                    iter->table++;  
                    iter->index = 0;  
                    ht = &iter->d->ht[1];  
                } else {  
                    break;  
                }  
            }  
            //根据下标选择集合点  
            iter->entry = ht->table[iter->index];  
        } else {  
            iter->entry = iter->nextEntry;  
        }  
        if (iter->entry) {  
            /* We need to save the 'next' here, the iterator user
             * may delete the entry we are returning. */  
            iter->nextEntry = iter->entry->next;  
            return iter->entry;  
        }  
    }  
    return NULL;  
}  
  
/* 释放迭代器 */  
void dictReleaseIterator(dictIterator *iter)  
{  
    if (!(iter->index == -1 && iter->table == 0)) {  
        if (iter->safe)  
            iter->d->iterators--;  
        else  
            //这时判断指纹是否还是之前定义的那个  
            assert(iter->fingerprint == dictFingerprint(iter->d));  
    }  
    zfree(iter);  
}  
  
/* Return a random entry from the hash table. Useful to
* implement randomized algorithms */  
/* 随机获取一个集合点 */  
dictEntry *dictGetRandomKey(dict *d)  
{  
    dictEntry *he, *orighe;  
    unsigned int h;  
    int listlen, listele;  
  
    if (dictSize(d) == 0) return NULL;  
    if (dictIsRehashing(d)) _dictRehashStep(d);  
    if (dictIsRehashing(d)) {  
        do {  
            //随机数向2个表格的总数求余运算  
            h = random() % (d->ht[0].size+d->ht[1].size);  
            he = (h >= d->ht[0].size) ? d->ht[1].table[h - d->ht[0].size] :  
                                      d->ht[0].table[h];  
        } while(he == NULL);  
    } else {  
        do {  
            h = random() & d->ht[0].sizemask;  
            he = d->ht[0].table[h];  
        } while(he == NULL);  
    }  
  
    /* Now we found a non empty bucket, but it is a linked
     * list and we need to get a random element from the list.
     * The only sane way to do so is counting the elements and
     * select a random index. */  
    listlen = 0;  
    orighe = he;  
    while(he) {  
        he = he->next;  
        listlen++;  
    }  
    listele = random() % listlen;  
    he = orighe;  
    while(listele--) he = he->next;  
    return he;  
}  
  
/* Function to reverse bits. Algorithm from:
* http://graphics.stanford.edu/~seander/bithacks.html#ReverseParallel */  
/* 很神奇的翻转位 */  
static unsigned long rev(unsigned long v) {  
    unsigned long s = 8 * sizeof(v); // bit size; must be power of 2  
    unsigned long mask = ~0;  
    while ((s >>= 1) > 0) {  
        mask ^= (mask << s);  
        v = ((v >> s) & mask) | ((v << s) & ~mask);  
    }  
    return v;  
}  
  
/* dictScan() is used to iterate over the elements of a dictionary.
*
* Iterating works in the following way:
*
* 1) Initially you call the function using a cursor (v) value of 0.
* 2) The function performs one step of the iteration, and returns the
*    new cursor value that you must use in the next call.
* 3) When the returned cursor is 0, the iteration is complete.
*
* The function guarantees that all the elements that are present in the
* dictionary from the start to the end of the iteration are returned.
* However it is possible that some element is returned multiple time.
*
* For every element returned, the callback 'fn' passed as argument is
* called, with 'privdata' as first argument and the dictionar entry
* 'de' as second argument.
*
* HOW IT WORKS.
*
* The algorithm used in the iteration was designed by Pieter Noordhuis.
* The main idea is to increment a cursor starting from the higher order
* bits, that is, instead of incrementing the cursor normally, the bits
* of the cursor are reversed, then the cursor is incremented, and finally
* the bits are reversed again.
*
* This strategy is needed because the hash table may be resized from one
* call to the other call of the same iteration.
*
* dict.c hash tables are always power of two in size, and they
* use chaining, so the position of an element in a given table is given
* always by computing the bitwise AND between Hash(key) and SIZE-1
* (where SIZE-1 is always the mask that is equivalent to taking the rest
*  of the division between the Hash of the key and SIZE).
*
* For example if the current hash table size is 16, the mask is
* (in binary) 1111. The position of a key in the hash table will be always
* the last four bits of the hash output, and so forth.
*
* WHAT HAPPENS IF THE TABLE CHANGES IN SIZE?
*
* If the hash table grows, elements can go anyway in one multiple of
* the old bucket: for example let's say that we already iterated with
* a 4 bit cursor 1100, since the mask is 1111 (hash table size = 16).
*
* If the hash table will be resized to 64 elements, and the new mask will
* be 111111, the new buckets that you obtain substituting in ??1100
* either 0 or 1, can be targeted only by keys that we already visited
* when scanning the bucket 1100 in the smaller hash table.
*
* By iterating the higher bits first, because of the inverted counter, the
* cursor does not need to restart if the table size gets bigger, and will
* just continue iterating with cursors that don't have '1100' at the end,
* nor any other combination of final 4 bits already explored.
*
* Similarly when the table size shrinks over time, for example going from
* 16 to 8, If a combination of the lower three bits (the mask for size 8
* is 111) was already completely explored, it will not be visited again
* as we are sure that, we tried for example, both 0111 and 1111 (all the
* variations of the higher bit) so we don't need to test it again.
*
* WAIT... YOU HAVE *TWO* TABLES DURING REHASHING!
*
* Yes, this is true, but we always iterate the smaller one of the tables,
* testing also all the expansions of the current cursor into the larger
* table. So for example if the current cursor is 101 and we also have a
* larger table of size 16, we also test (0)101 and (1)101 inside the larger
* table. This reduces the problem back to having only one table, where
* the larger one, if exists, is just an expansion of the smaller one.
*
* LIMITATIONS
*
* This iterator is completely stateless, and this is a huge advantage,
* including no additional memory used.
*
* The disadvantages resulting from this design are:
*
* 1) It is possible that we return duplicated elements. However this is usually
*    easy to deal with in the application level.
* 2) The iterator must return multiple elements per call, as it needs to always
*    return all the keys chained in a given bucket, and all the expansions, so
*    we are sure we don't miss keys moving.
* 3) The reverse cursor is somewhat hard to understand at first, but this
*    comment is supposed to help.
*/  
/* 扫描方法 */  
unsigned long dictScan(dict *d,  
                       unsigned long v,  
                       dictScanFunction *fn,  
                       void *privdata)  
{  
    dictht *t0, *t1;  
    const dictEntry *de;  
    unsigned long m0, m1;  
  
    if (dictSize(d) == 0) return 0;  
  
    if (!dictIsRehashing(d)) {  
        t0 = &(d->ht[0]);  
        m0 = t0->sizemask;  
  
        /* Emit entries at cursor */  
        de = t0->table[v & m0];  
        while (de) {  
            fn(privdata, de);  
            de = de->next;  
        }  
  
    } else {  
        t0 = &d->ht[0];  
        t1 = &d->ht[1];  
  
        /* Make sure t0 is the smaller and t1 is the bigger table */  
        if (t0->size > t1->size) {  
            t0 = &d->ht[1];  
            t1 = &d->ht[0];  
        }  
  
        m0 = t0->sizemask;  
        m1 = t1->sizemask;  
  
        /* Emit entries at cursor */  
        de = t0->table[v & m0];  
        while (de) {  
            fn(privdata, de);  
            de = de->next;  
        }  
  
        /* Iterate over indices in larger table that are the expansion
         * of the index pointed to by the cursor in the smaller table */  
        do {  
            /* Emit entries at cursor */  
            de = t1->table[v & m1];  
            while (de) {  
                fn(privdata, de);  
                de = de->next;  
            }  
  
            /* Increment bits not covered by the smaller mask */  
            v = (((v | m0) + 1) & ~m0) | (v & m0);  
  
            /* Continue while bits covered by mask difference is non-zero */  
        } while (v & (m0 ^ m1));  
    }  
  
    /* Set unmasked bits so incrementing the reversed cursor
     * operates on the masked bits of the smaller table */  
    v |= ~m0;  
  
    /* Increment the reverse cursor */  
    v = rev(v);  
    v++;  
    v = rev(v);  
  
    return v;  
}  
  
/* ------------------------- private functions ------------------------------ */  
  
/* Expand the hash table if needed */  
/* 判断是否需要扩容 */  
static int _dictExpandIfNeeded(dict *d)  
{  
    /* Incremental rehashing already in progress. Return. */  
    if (dictIsRehashing(d)) return DICT_OK;  
  
    /* If the hash table is empty expand it to the initial size. */  
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);  
  
    /* If we reached the 1:1 ratio, and we are allowed to resize the hash
     * table (global setting) or we should avoid it but the ratio between
     * elements/buckets is over the "safe" threshold, we resize doubling
     * the number of buckets. */  
    /* 判断是否需要扩容 */  
    if (d->ht[0].used >= d->ht[0].size &&  
        (dict_can_resize ||  
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))  
    {  
        return dictExpand(d, d->ht[0].used*2);  
    }  
    return DICT_OK;  
}  
  
/* Our hash table capability is a power of two */  
/* 哈希表的容量以2的幂次方，所以数量以2的幂次向上取 */  
static unsigned long _dictNextPower(unsigned long size)  
{  
    unsigned long i = DICT_HT_INITIAL_SIZE;  
  
    if (size >= LONG_MAX) return LONG_MAX;  
    while(1) {  
        if (i >= size)  
            return i;  
        i *= 2;  
    }  
}  
  
/* Returns the index of a free slot that can be populated with
* a hash entry for the given 'key'.
* If the key already exists, -1 is returned.
*
* Note that if we are in the process of rehashing the hash table, the
* index is always returned in the context of the second (new) hash table. */  
/* 获取key值对应的哈希索引值，如果已经存在此key则返回-1 */  
static int _dictKeyIndex(dict *d, const void *key)  
{  
    unsigned int h, idx, table;  
    dictEntry *he;  
  
    /* Expand the hash table if needed */  
    if (_dictExpandIfNeeded(d) == DICT_ERR)  
        return -1;  
    /* Compute the key hash value */  
    h = dictHashKey(d, key);  
    for (table = 0; table <= 1; table++) {  
        idx = h & d->ht[table].sizemask;  
        /* Search if this slot does not already contain the given key */  
        he = d->ht[table].table[idx];  
        while(he) {  
            if (dictCompareKeys(d, key, he->key))  
                return -1;  
            he = he->next;  
        }  
        if (!dictIsRehashing(d)) break;  
    }  
    return idx;  
}  
  
/* 清空整个字典，即清空里面的2张哈希表 */  
void dictEmpty(dict *d, void(callback)(void*)) {  
    _dictClear(d,&d->ht[0],callback);  
    _dictClear(d,&d->ht[1],callback);  
    d->rehashidx = -1;  
    d->iterators = 0;  
}  
  
/*启用哈希表调整*/  
void dictEnableResize(void) {  
    dict_can_resize = 1;  
}  
  
/* 启用哈希表调整 */  
void dictDisableResize(void) {  
    dict_can_resize = 0;  
}  
  
#if 0  
  
/* The following is code that we don't use for Redis currently, but that is part
of the library. */  
/* redis中还存着调试的代码 */  
/* ----------------------- Debugging ------------------------*/  
  
#define DICT_STATS_VECTLEN 50  
static void _dictPrintStatsHt(dictht *ht) {  
    unsigned long i, slots = 0, chainlen, maxchainlen = 0;  
    unsigned long totchainlen = 0;  
    unsigned long clvector[DICT_STATS_VECTLEN];  
  
    if (ht->used == 0) {  
        printf("No stats available for empty dictionaries\n");  
        return;  
    }  
  
    for (i = 0; i < DICT_STATS_VECTLEN; i++) clvector[i] = 0;  
    for (i = 0; i < ht->size; i++) {  
        dictEntry *he;  
  
        if (ht->table[i] == NULL) {  
            clvector[0]++;  
            continue;  
        }  
        slots++;  
        /* For each hash entry on this slot... */  
        chainlen = 0;  
        he = ht->table[i];  
        while(he) {  
            chainlen++;  
            he = he->next;  
        }  
        clvector[(chainlen < DICT_STATS_VECTLEN) ? chainlen : (DICT_STATS_VECTLEN-1)]++;  
        if (chainlen > maxchainlen) maxchainlen = chainlen;  
        totchainlen += chainlen;  
    }  
    printf("Hash table stats:\n");  
    printf(" table size: %ld\n", ht->size);  
    printf(" number of elements: %ld\n", ht->used);  
    printf(" different slots: %ld\n", slots);  
    printf(" max chain length: %ld\n", maxchainlen);  
    printf(" avg chain length (counted): %.02f\n", (float)totchainlen/slots);  
    printf(" avg chain length (computed): %.02f\n", (float)ht->used/slots);  
    printf(" Chain length distribution:\n");  
    for (i = 0; i < DICT_STATS_VECTLEN-1; i++) {  
        if (clvector[i] == 0) continue;  
        printf("   %s%ld: %ld (%.02f%%)\n",(i == DICT_STATS_VECTLEN-1)?">= ":"", i, clvector[i], ((float)clvector[i]/ht->size)*100);  
    }  
}  
  
void dictPrintStats(dict *d) {  
    _dictPrintStatsHt(&d->ht[0]);  
    if (dictIsRehashing(d)) {  
        printf("-- Rehashing into ht[1]:\n");  
        _dictPrintStatsHt(&d->ht[1]);  
    }  
}  
  
/* ----------------------- StringCopy Hash Table Type ------------------------*/  
  
static unsigned int _dictStringCopyHTHashFunction(const void *key)  
{  
    return dictGenHashFunction(key, strlen(key));  
}  
  
static void *_dictStringDup(void *privdata, const void *key)  
{  
    int len = strlen(key);  
    char *copy = zmalloc(len+1);  
    DICT_NOTUSED(privdata);  
  
    memcpy(copy, key, len);  
    copy[len] = '\0';  
    return copy;  
}  
  
static int _dictStringCopyHTKeyCompare(void *privdata, const void *key1,  
        const void *key2)  
{  
    DICT_NOTUSED(privdata);  
  
    return strcmp(key1, key2) == 0;  
}  
  
static void _dictStringDestructor(void *privdata, void *key)  
{  
    DICT_NOTUSED(privdata);  
  
    zfree(key);  
}  
  
/* 定义了3种类型的dictType，有些类型无val dup方法的定义 */  
dictType dictTypeHeapStringCopyKey = {  
    _dictStringCopyHTHashFunction, /* hash function */  
    _dictStringDup,                /* key dup */  
    NULL,                          /* val dup */  
    _dictStringCopyHTKeyCompare,   /* key compare */  
    _dictStringDestructor,         /* key destructor */  
    NULL                           /* val destructor */  
};  
  
/* This is like StringCopy but does not auto-duplicate the key.
* It's used for intepreter's shared strings. */  
dictType dictTypeHeapStrings = {  
    _dictStringCopyHTHashFunction, /* hash function */  
    NULL,                          /* key dup */  
    NULL,                          /* val dup */  
    _dictStringCopyHTKeyCompare,   /* key compare */  
    _dictStringDestructor,         /* key destructor */  
    NULL                           /* val destructor */  
};  
  
/* This is like StringCopy but also automatically handle dynamic
* allocated C strings as values. */  
dictType dictTypeHeapStringCopyKeyValue = {  
    _dictStringCopyHTHashFunction, /* hash function */  
    _dictStringDup,                /* key dup */  
    _dictStringDup,                /* val dup */  
    _dictStringCopyHTKeyCompare,   /* key compare */  
    _dictStringDestructor,         /* key destructor */  
    _dictStringDestructor,         /* val destructor */  
};  
#endif 
```

