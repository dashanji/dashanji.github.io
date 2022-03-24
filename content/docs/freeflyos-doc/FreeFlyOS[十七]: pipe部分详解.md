---
title: Chapter-17 pipe部分详解
description: Chapter 17 of FreeFlyOS
toc: true
authors:
tags:
weight: 17
categories:
series:
date: '2022-03-24'
lastmod: '2022-03-24'
draft: false
---
## ioqueue.c

```
#include "ioqueue.h"
#include "../task/task.h"
#include "../debug/debug.h"
extern struct task_struct *current;
/* 初始化io队列ioq */
void ioqueue_init(struct ioqueue* ioq) {
   lock_init(&ioq->lock);     // 初始化io队列的锁
   ioq->producer = ioq->consumer = NULL;  // 生产者和消费者置空
   ioq->head = ioq->tail = 0; // 队列的首尾指针指向缓冲区数组第0个位置
}

/* 返回pos在缓冲区中的下一个位置值 */
static int next_pos(int pos) {
   return (pos + 1) % bufsize; 
}

/* 判断队列是否已满 */
char ioq_full(struct ioqueue* ioq) {
   //ASSERT(get_now_intr_status() == INTR_OFF);
   return next_pos(ioq->head) == ioq->tail;
}

/* 判断队列是否已空 */
static char ioq_empty(struct ioqueue* ioq) {
   //ASSERT(get_now_intr_status() == INTR_OFF);
   return ioq->head == ioq->tail;
}

/* 使当前生产者或消费者在此缓冲区上等待 */
static void ioq_wait(struct task_struct** waiter) {
   ASSERT(*waiter == NULL && waiter != NULL);
   *waiter = current;
   thread_block(STOPPED);
}

/* 唤醒waiter */
static void wakeup(struct task_struct** waiter) {
   ASSERT(*waiter != NULL);
   thread_unblock(*waiter); 
   *waiter = NULL;
}

/* 消费者从ioq队列中获取一个字符 */
char ioq_getchar(struct ioqueue* ioq) {
  //ASSERT(get_now_intr_status() == INTR_OFF);

/* 若缓冲区(队列)为空,把消费者ioq->consumer记为当前线程自己,
 * 目的是将来生产者往缓冲区里装商品后,生产者知道唤醒哪个消费者,
 * 也就是唤醒当前线程自己*/
   while (ioq_empty(ioq)) {
      lock_acquire(&ioq->lock);	 
      ioq_wait(&ioq->consumer);
      lock_release(&ioq->lock);
   }

   char byte = ioq->buf[ioq->tail];	  // 从缓冲区中取出
   ioq->tail = next_pos(ioq->tail);	  // 把读游标移到下一位置

   if (ioq->producer != NULL) {
      wakeup(&ioq->producer);		  // 唤醒生产者
   }

   return byte; 
}

/* 生产者往ioq队列中写入一个字符byte */
void ioq_putchar(struct ioqueue* ioq, char byte) {
   //ASSERT(get_now_intr_status() == INTR_OFF);

/* 若缓冲区(队列)已经满了,把生产者ioq->producer记为自己,
 * 为的是当缓冲区里的东西被消费者取完后让消费者知道唤醒哪个生产者,
 * 也就是唤醒当前线程自己*/
   while (ioq_full(ioq)) {
      lock_acquire(&ioq->lock);
      ioq_wait(&ioq->producer);
      lock_release(&ioq->lock);
   }
   ioq->buf[ioq->head] = byte;      // 把字节放入缓冲区中
   ioq->head = next_pos(ioq->head); // 把写游标移到下一位置

   if (ioq->consumer != NULL) {
      wakeup(&ioq->consumer);          // 唤醒消费者
   }
}

/* 返回环形缓冲区中的数据长度 */
unsigned int ioq_length(struct ioqueue* ioq) {
   unsigned int len = 0;
   if (ioq->head >= ioq->tail) {
      len = ioq->head - ioq->tail;
   } else {
      len = bufsize - (ioq->tail - ioq->head);     
   }
   return len;
}

```



## ioqueue.h

```
#ifndef _IOQUEUE_H_
#define _IOQUEUE_H_
#include "../sync/sync.h"
#include "../task/task.h"

#define bufsize 64

/* 环形队列 */
struct ioqueue {
// 生产者消费者问题
    struct lock lock;
 /* 生产者,缓冲区不满时就继续往里面放数据,
  * 否则就睡眠,此项记录哪个生产者在此缓冲区上睡眠。*/
    struct task_struct* producer;

 /* 消费者,缓冲区不空时就继续从往里面拿数据,
  * 否则就睡眠,此项记录哪个消费者在此缓冲区上睡眠。*/
    struct task_struct* consumer;
    char buf[bufsize];			    // 缓冲区大小
    int head;			    // 队首,数据往队首处写入
    int tail;			    // 队尾,数据从队尾处读出
};

void ioqueue_init(struct ioqueue* ioq);
char ioq_full(struct ioqueue* ioq);
char ioq_getchar(struct ioqueue* ioq);
void ioq_putchar(struct ioqueue* ioq, char byte);
unsigned int ioq_length(struct ioqueue* ioq);
#endif

```



## pipe.c

```
#include "pipe.h"
#include "ioqueue.h"
#include "../file/file.h"
#include "../file/fs.h"
#include "../mem/vmm.h"
#define NULL ((void *)0)
extern struct file file_table[MAX_FILE_OPEN];
/* 判断文件描述符local_fd是否是管道 */
char is_pipe(unsigned int local_fd) {
   unsigned int global_fd = fd_local2global(local_fd); 
   return file_table[global_fd].fd_flag == PIPE_FLAG;
}

/* 创建管道,成功返回0,失败返回-1 */
int sys_pipe(int pipefd[2]) {
   int global_fd = get_free_slot_in_global();

   /* 申请一页内核内存做环形缓冲区 */
   file_table[global_fd].fd_inode =vmm_malloc(VMM_PAGE_SIZE,1); 

   /* 初始化环形缓冲区 */
   ioqueue_init((struct ioqueue*)file_table[global_fd].fd_inode);
   if (file_table[global_fd].fd_inode == NULL) {
      return -1;
   }
  
   /* 将fd_flag复用为管道标志 */
   file_table[global_fd].fd_flag = PIPE_FLAG;

   /* 将fd_pos复用为管道打开数 */
   file_table[global_fd].fd_pos = 2;
   pipefd[0] = task_fd_install(global_fd);
   pipefd[1] = task_fd_install(global_fd);
   return 0;
}

/* 从管道中读数据 */
unsigned int pipe_read(int fd, void* buf, unsigned int count) {
   char* buffer = buf;
   unsigned int bytes_read = 0;
   unsigned int global_fd = fd_local2global(fd);

   /* 获取管道的环形缓冲区 */
   struct ioqueue* ioq = (struct ioqueue*)file_table[global_fd].fd_inode;

   /* 选择较小的数据读取量,避免阻塞 */
   unsigned int ioq_len = ioq_length(ioq);
   unsigned int size = ioq_len > count ? count : ioq_len;
   while (bytes_read < size) {
      *buffer = ioq_getchar(ioq);
      bytes_read++;
      buffer++;
   }
   return bytes_read;
}

/* 往管道中写数据 */
unsigned int pipe_write(int fd, const void* buf, unsigned int count) {
   unsigned int bytes_write = 0;
   unsigned int global_fd = fd_local2global(fd);
   struct ioqueue* ioq = (struct ioqueue*)file_table[global_fd].fd_inode;

   /* 选择较小的数据写入量,避免阻塞 */
   unsigned int ioq_left = bufsize - ioq_length(ioq);
   unsigned int size = ioq_left > count ? count : ioq_left;

   const char* buffer = buf;
   while (bytes_write < size) {
      ioq_putchar(ioq, *buffer);
      bytes_write++;
      buffer++;
   }
   return bytes_write;
}

```



## pipe.h

```
#ifndef _PIPE_H_
#define _PIPE_H_

#define PIPE_FLAG 0xFFFF
char is_pipe(unsigned int local_fd);
int sys_pipe(int pipefd[2]);
unsigned int pipe_read(int fd, void* buf, unsigned int count);
unsigned int pipe_write(int fd, const void* buf, unsigned int count);
#endif

```

该目录主要是双向管道的实现，很像一个生产者和消费者问题，主要步骤如下：

1、初始化一个IO队列，主要包含了一个锁和一个缓冲区队列，进程通过该IO队列进行通信。

2、管道本质上是文件描述符表对应的i节点，对于一个双向管道，需要有读管道和写管道，将其对应到全局文件表中，之后将其分别安装在进程自己的文件描述符表中。

3、父子进程通过自己的文件描述符表信息可以获取全局文件表的i节点，从而进行通信，缓冲区有数据则唤醒消费者，若无数据则消费者等待。


