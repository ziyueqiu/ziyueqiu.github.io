---
layout:     post
title:      Experiences with CLHT, a research concurrent hash table from EPFL
date:       2022-02-02
tags:
    - Hash Table
categories: caching
comments: true
---

## What you will see

1. Fix CLHT into use

2. Add `clht_clear()` to support usage

View complete code change from my [forked repo here](https://github.com/ziyueqiu/CLHT)

## Code change for usage

### Machine dependent parameters

Get info from machine (for example)

~~~bash
$ cat /proc/cpuinfo | grep name | cut -f2 -d: | uniq -c
160  Intel(R) Xeon(R) Platinum 8380 CPU @ 2.30GHz

...
~~~



Add info into `external/include/utils.h` (for example)

```c
#define XEONR
  
...
  //machine dependent parameters
#ifdef __sparc__ // prior
...
  
#elif defined(XEONR)
#define NUMBER_OF_SOCKETS 2
#define CORES_PER_SOCKET 40
#define CACHE_LINE_SIZE 64
#define NOP_DURATION 2

  static uint8_t __attribute__ ((unused)) the_cores[] = {
    0, 1, 2, 3, 4, 5, 6, 7, 8, 9,
    10, 11, 12, 13, 14, 15, 16, 17, 18, 19,
    20, 21, 22, 23, 24, 25, 26, 27, 28, 29,
    30, 31, 32, 33, 34, 35, 36, 37, 38, 39,
    40, 41, 42, 43, 44, 45, 46, 47, 48, 49,
    50, 51, 52, 53, 54, 55, 56, 57, 58, 59,
    60, 61, 62, 63, 64, 65, 66, 67, 68, 69,
    70, 71, 72, 73, 74, 75, 76, 77, 78, 79,
  };
  static uint8_t __attribute__ ((unused)) the_sockets[] =
  {
    0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
    1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
    1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
    1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
    1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
  };
```

### C++ support

```c
#ifdef __cplusplus
extern "C"
{
#endif
  
... // prior declaration
  
#ifdef __cplusplus
}
#endif
```

## clht_clear

for lock based version `src/clht_lb.c` or lock based & resizing version `src/clht_lb_res.c`

we implemented `clht_clear` since CLHT only supports `clht_destroy` before

```c
void
clht_clear(clht_hashtable_t* hashtable)
{
  uint64_t num_buckets = hashtable->num_buckets;
  bucket_t* bucket;

  printf("Clear number of buckets: %u\n", num_buckets);
  uint64_t bin;
  for (bin = 0; bin < num_buckets; bin++)
  {
      bucket = hashtable->table + bin;
      uint32_t j;
      do
			{
        for (j = 0; j < ENTRIES_PER_BUCKET; j++)
        {
          bucket->key[j] = 0;
        }
        bucket = bucket->next;
      }
      while (bucket != NULL);
  }
}
```

compared with

```c
void
clht_destroy(clht_hashtable_t* hashtable)
{
  free(hashtable->table);
  free(hashtable);
}
```

## Reference

1. Related Publication: [*Asynchronized Concurrency: The Secret to Scaling Concurrent Search Data Structures*](https://infoscience.epfl.ch/record/207109), Tudor David, Rachid Guerraoui, Vasileios Trigonakis (alphabetical order), ASPLOS '15

