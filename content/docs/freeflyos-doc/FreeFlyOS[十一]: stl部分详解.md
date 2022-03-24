---
title: Chapter-11 stl部分详解
description: Chapter 11 of FreeFlyOS
toc: true
authors:
tags:
weight: 11
categories:
series:
date: '2022-03-24'
lastmod: '2022-03-24'
draft: false
---
## list.h

链表数据结构

```
#ifndef _LIST_H_
#define _LIST_H_
/* *
 * Simple doubly linked list implementation.
 *
 * Some of the internal functions ("__xxx") are useful when manipulating
 * whole lists rather than single entries, as sometimes we already know
 * the next/prev entries and we can generate better code by using them
 * directly rather than using the generic single-entry routines.
 * */

struct list_entry {
    struct list_entry *prev, *next;
};

typedef struct list_entry list_entry_t;

static inline void list_init(list_entry_t *elm) __attribute__((always_inline));
static inline void list_add(list_entry_t *listelm, list_entry_t *elm) __attribute__((always_inline));
static inline void list_add_before(list_entry_t *listelm, list_entry_t *elm) __attribute__((always_inline));
static inline void list_add_after(list_entry_t *listelm, list_entry_t *elm) __attribute__((always_inline));
static inline void list_del(list_entry_t *listelm) __attribute__((always_inline));
static inline void list_del_init(list_entry_t *listelm) __attribute__((always_inline));
static inline char list_empty(list_entry_t *list) __attribute__((always_inline));
static inline list_entry_t *list_next(list_entry_t *listelm) __attribute__((always_inline));
static inline list_entry_t *list_prev(list_entry_t *listelm) __attribute__((always_inline));

static inline void __list_add(list_entry_t *elm, list_entry_t *prev, list_entry_t *next) __attribute__((always_inline));
static inline void __list_del(list_entry_t *prev, list_entry_t *next) __attribute__((always_inline));

/* *
 * list_init - initialize a new entry
 * @elm:        new entry to be initialized
 * */
static inline void
list_init(list_entry_t *elm) {
    elm->prev = elm->next = elm;
}

/* *
 * list_add - add a new entry
 * @listelm:    list head to add after
 * @elm:        new entry to be added
 *
 * Insert the new element @elm *after* the element @listelm which
 * is already in the list.
 * */
static inline void
list_add(list_entry_t *listelm, list_entry_t *elm) {
    list_add_after(listelm, elm);
}

/* *
 * list_add_before - add a new entry
 * @listelm:    list head to add before
 * @elm:        new entry to be added
 *
 * Insert the new element @elm *before* the element @listelm which
 * is already in the list.
 * */
static inline void
list_add_before(list_entry_t *listelm, list_entry_t *elm) {
    __list_add(elm, listelm->prev, listelm);
}

/* *
 * list_add_after - add a new entry
 * @listelm:    list head to add after
 * @elm:        new entry to be added
 *
 * Insert the new element @elm *after* the element @listelm which
 * is already in the list.
 * */
static inline void
list_add_after(list_entry_t *listelm, list_entry_t *elm) {
    __list_add(elm, listelm, listelm->next);
}

/* *
 * list_del - deletes entry from list
 * @listelm:    the element to delete from the list
 *
 * Note: list_empty() on @listelm does not return true after this, the entry is
 * in an undefined state.
 * */
static inline void
list_del(list_entry_t *listelm) {
    __list_del(listelm->prev, listelm->next);
}

/* *
 * list_del_init - deletes entry from list and reinitialize it.
 * @listelm:    the element to delete from the list.
 *
 * Note: list_empty() on @listelm returns true after this.
 * */
static inline void
list_del_init(list_entry_t *listelm) {
    list_del(listelm);
    list_init(listelm);
}

/* *
 * list_empty - tests whether a list is empty
 * @list:       the list to test.
 * */
static inline char
list_empty(list_entry_t *list) {
    if(list->next == list)  
        return 1;
    return 0;
}

/* *
 * list_next - get the next entry
 * @listelm:    the list head
 **/
static inline list_entry_t *
list_next(list_entry_t *listelm) {
    return listelm->next;
}

/* *
 * list_prev - get the previous entry
 * @listelm:    the list head
 **/
static inline list_entry_t *
list_prev(list_entry_t *listelm) {
    return listelm->prev;
}

/* *
 * Insert a new entry between two known consecutive entries.
 *
 * This is only for internal list manipulation where we know
 * the prev/next entries already!
 * */
static inline void
__list_add(list_entry_t *elm, list_entry_t *prev, list_entry_t *next) {
    prev->next = next->prev = elm;
    elm->next = next;
    elm->prev = prev;
}

/* *
 * Delete a list entry by making the prev/next entries point to each other.
 *
 * This is only for internal list manipulation where we know
 * the prev/next entries already!
 * */
static inline void
__list_del(list_entry_t *prev, list_entry_t *next) {
    prev->next = next;
    next->prev = prev;
}


#endif /* !_LIST_H_ */


```

## defs,h

```
#ifndef _DEFS_H_
#define _DEFS_H_


/* Return the offset of 'member' relative to the beginning of a struct type */
#define offsetof(type, member)                                      \
    ((unsigned int)(&((type *)0)->member))

/* *
 * to_struct - get the struct from a ptr
 * @ptr:    a struct pointer of member
 * @type:   the type of the struct this is embedded in
 * @member: the name of the member within the struct
 * */
//char * =unsigned int
#define to_struct(ptr, type, member)                               \
    ((type *)((char *)(ptr) - offsetof(type, member)))

//将一个字节某位设置为flag
#define set_char_bit(c,offset,flag) c&(flag<<offset|0xFF) 


#endif
```



## elf.h

```
#ifndef _ELF_H_
#define _ELF_H_
/* file header */
struct elfhdr {
    unsigned int e_magic;     // must equal ELF_MAGIC
    unsigned char e_elf[12];
    unsigned short e_type;      // 1=relocatable, 2=executable, 3=shared object, 4=core image
    unsigned short e_machine;   // 3=x86, 4=68K, etc.
    unsigned int e_version;   // file version, always 1
    unsigned int e_entry;     // entry point if executable
    unsigned int e_phoff;     // file position of program header or 0
    unsigned int e_shoff;     // file position of section header or 0
    unsigned int e_flags;     // architecture-specific flags, usually 0
    unsigned short e_ehsize;    // size of this elf header
    unsigned short e_phentsize; // size of an entry in program header
    unsigned short e_phnum;     // number of entries in program header or 0
    unsigned short e_shentsize; // size of an entry in section header
    unsigned short e_shnum;     // number of entries in section header or 0
    unsigned short e_shstrndx;  // section number that contains section name strings
};

/* program section header */
struct proghdr {
    unsigned int p_type;   // loadable code or data, dynamic linking info,etc.
    unsigned int p_offset; // file offset of segment
    unsigned int p_va;     // virtual address to map segment
    unsigned int p_pa;     // physical address, not used
    unsigned int p_filesz; // size of segment in file
    unsigned int p_memsz;  // size of segment in memory (bigger if contains bss）
    unsigned int p_flags;  // read/write/execute bits
    unsigned int p_align;  // required alignment, invariably hardware page size
};
#endif
```



## hash.h

```
#ifndef _HASH_H_
#define _HASH_H_

/* 2^31 + 2^29 - 2^25 + 2^22 - 2^19 - 2^16 + 1 */
//Knuth 建议把这个参数取到最接近 2^32 黄金分割的素数，然后就有这个 Magic Number 了。
#define GOLDEN_RATIO_PRIME_32       0x9e370001UL

/* *
 * hash32 - generate a hash value in the range [0, 2^@bits - 1]
 * @val:    the input value
 * @bits:   the number of bits in a return value
 *
 * High bits are more random, so we use them.
 * */
static unsigned int
hash32(unsigned int val, unsigned int bits) {
    unsigned int hash = val * GOLDEN_RATIO_PRIME_32;
    return (hash >> (32 - bits));
}

#endif
```

这个目录主要就是一些常见的数据结构，以及转换的小技巧。

包含链表、哈希表、elf文件以及已知结构体成员如何获取结构体等。


