---
title: Chapter-19 user部分详解
description: Chapter 19 of FreeFlyOS
toc: true
authors:
tags:
weight: 19
categories:
series:
date: '2022-03-24'
lastmod: '2022-03-24'
draft: false
---
## stdio.c

```
#include "stdio.h"
#include "va_list.h"
#include "user_syscall.h"

void printf(char *fmt,...){
    user_va_list ap;
    
    char c;
    char *str;

    int dec_num;
    int hex_num;

    unsigned int unsigned_dec_num;
    unsigned int unsigned_hex_num;

    long long ll_hex_num;

    unsigned long long ull_hex_num;

    char bits=0;     //record the number's bits

    user_va_start(ap,fmt);

    while(*fmt){
        if(*fmt=='%'){
user_dis_num:    
            switch (*(++fmt))
            {
                case 'c':
                    c=user_va_arg(ap,char);
                    user_print_char(c);
                    break;
                case 's':
                    str=user_va_arg(ap,char *);
                    user_print_string(str);
                    break;
                case 'd':
                    dec_num=user_va_arg(ap,int);
                    if(bits){
                        user_print_num(dec_num,dec,bits,display_bits);
                    }
                    else{
                        user_print_num(dec_num,dec,ulonglong_max,display_num);
                    }
                    break;
                case 'X':
                case 'x':
                    hex_num=user_va_arg(ap,int);
                    if(bits){
                        user_print_num(hex_num,hex,bits,display_bits);
                    }
                    else{
                        user_print_num(hex_num,hex,ulonglong_max,display_num);
                    }
                    break;
                case 'l':
                case 'L':
                    ll_hex_num=user_va_arg(ap,long long);
                    if(bits){
                        user_print_num(ll_hex_num,hex,bits,display_bits);
                    }
                    else{
                        user_print_num(ll_hex_num,hex,ulonglong_max,display_num);
                    }
                    break;
                case 'u':
                    switch (*(++fmt))
                    {
                        case 'd':
                            unsigned_dec_num=user_va_arg(ap,unsigned int);
                            if(bits){
                                user_print_num(unsigned_dec_num,dec,bits,display_bits);
                            }
                            else{
                                user_print_num(unsigned_dec_num,dec,ulonglong_max,display_num);
                            }
                            break;
                        case 'X':
                        case 'x':
                            unsigned_hex_num=user_va_arg(ap,unsigned int);
                            if(bits){
                                user_print_num(unsigned_hex_num,hex,bits,display_bits);
                            }
                            else{
                                user_print_num(unsigned_hex_num,hex,ulonglong_max,display_num);
                            }
                            break;
                        case 'l':
                        case 'L':
                            ull_hex_num=user_va_arg(ap,unsigned long long);
                            if(bits){
                                user_print_num(ull_hex_num,hex,bits,display_bits);
                            }
                            else{
                                user_print_num(ull_hex_num,hex,ulonglong_max,display_num);
                                }
                            break;
                        default:
                            break;
                    }
                    break;
                //read the bits of number displayed,the range is 00-99
                case '0':
                case '1':
                case '2':
                case '3':
                case '4':
                case '5':
                case '6':
                case '7':
                case '8':
                case '9':
                    bits=(*fmt-'0')*10+(*(++fmt)-'0');
                    goto user_dis_num;
                    break;
                default:    
                    user_print_string("error format!Please correct it!");
                    break;
            }
            fmt++;
        }
        else{
            user_print_char(*fmt);
            fmt++;
        }
        bits=0;
    }
}
```



## stdio.h

```
#ifndef _STDIO_H_
#define _STDIO_H_
//the max of unsigned char is 255 in dec(3 character)
#define uchar_max 3
//the max of unsigned short is 65535 in dec(5 character)
#define ushort_max 5
//the max of unsigned int is 4294967295 in dec(10 character)
#define int_max 10
//the max of unsigned long long is 18446744073709551615 in dec(20 character)
#define ulonglong_max 20


#define dec 10  //Decimal
#define hex 16  //Hexadecimal  

#define display_bits 1   //display x bits number     ex. 100->00100
#define display_num  0   //display only number bits  ex. 100->100  
//__attribute__( (section(".user.text") ) )  
void printf(char *fmt,...);
#endif
```



## string.c

```
#include "string.h"
#define NULL ((void *)0)
static inline void *
__memset(void *s, char c, unsigned int n) {
    int d0, d1;
    asm volatile (
        "rep; stosb;"
        : "=&c" (d0), "=&D" (d1)
        : "0" (n), "a" (c), "1" (s)
        : "memory");
    return s;
}

/* *
 * memset - sets the first @n bytes of the memory area pointed by @s
 * to the specified value @c.
 * @s:      pointer the the memory area to fill
 * @c:      value to set
 * @n:      number of bytes to be set to the value
 *
 * The memset() function returns @s.
 * */
void *
user_memset(void *s, char c, unsigned int n) {
    return __memset(s, c, n);
}
/* 将字符串src_拼接到dst_后,将回拼接的串地址 */
char* user_strcat(char* dst_, const char* src_) {
   char* str = dst_;
   while (*str++);
   --str;      // 别看错了，--str是独立的一句，并不是while的循环体
   while((*str++ = *src_++));	 // 当*str被赋值为0时,此时表达式不成立,正好添加了字符串结尾的0.
   return dst_;
}
static inline int
__strcmp(const char *s1, const char *s2) {
    int d0, d1, ret;
    asm volatile (
        "1: lodsb;"
        "scasb;"
        "jne 2f;"
        "testb %%al, %%al;"
        "jne 1b;"
        "xorl %%eax, %%eax;"
        "jmp 3f;"
        "2: sbbl %%eax, %%eax;"
        "orb $1, %%al;"
        "3:"
        : "=a" (ret), "=&S" (d0), "=&D" (d1)
        : "1" (s1), "2" (s2)
        : "memory");
    return ret;
}

/* *
 * strcmp - compares the string @s1 and @s2
 * @s1:     string to be compared
 * @s2:     string to be compared
 *
 * This function starts comparing the first character of each string. If
 * they are equal to each other, it continues with the following pairs until
 * the characters differ or until a terminanting null-character is reached.
 *
 * Returns an integral value indicating the relationship between the strings:
 * - A zero value indicates that both strings are equal;
 * - A value greater than zero indicates that the first character that does
 *   not match has a greater value in @s1 than in @s2;
 * - And a value less than zero indicates the opposite.
 * */
int
user_strcmp(const char *s1, const char *s2) {
    return __strcmp(s1, s2);
}
/* 从后往前查找字符串str中首次出现字符ch的地址(不是下标,是地址) */
char* user_strrchr(const char* str, const unsigned char ch) {
   const char* last_char = NULL;
   /* 从头到尾遍历一次,若存在ch字符,last_char总是该字符最后一次出现在串中的地址(不是下标,是地址)*/
   while (*str != 0) {
      if (*str == ch) {
	 last_char = str;
      }
      str++;
   }
   return (char*)last_char;
}
unsigned int
user_strlen(const char *s) {
    unsigned int cnt = 0;
    while (*s ++ != '\0') {
        cnt ++;
    }
    return cnt;
}
static inline void *
__memcpy(void *dst, const void *src, unsigned int n) {
    int d0, d1, d2;
    asm volatile (
        "rep; movsl;"
        "movl %4, %%ecx;"
        "andl $3, %%ecx;"
        "jz 1f;"
        "rep; movsb;"
        "1:"
        : "=&c" (d0), "=&D" (d1), "=&S" (d2)
        : "0" (n / 4), "g" (n), "1" (dst), "2" (src)
        : "memory");
    return dst;
}

/* *
 * memcpy - copies the value of @n bytes from the location pointed by @src to
 * the memory area pointed by @dst.
 * @dst     pointer to the destination array where the content is to be copied
 * @src     pointer to the source of data to by copied
 * @n:      number of bytes to copy
 *
 * The memcpy() returns @dst.
 *
 * Note that, the function does not check any terminating null character in @src,
 * it always copies exactly @n bytes. To avoid overflows, the size of arrays pointed
 * by both @src and @dst, should be at least @n bytes, and should not overlap
 * (for overlapping memory area, memmove is a safer approach).
 * */
void *
user_memcpy(void *dst, const void *src, unsigned int n) {
    return __memcpy(dst, src, n);
}
/* 将字符串从src_复制到dst_ */
char* user_strcpy(char* dst_, const char* src_) {
   char* r = dst_;		       // 用来返回目的字符串起始地址
   while((*dst_++ = *src_++));
   return r;
}
```



## string.h

```
#ifndef _STRING_H_
#define _STRING_H_
//__attribute__( (section(".user.text") ) )  
static inline void *__memset(void *s, char c, unsigned int n);
//__attribute__( (section(".user.text") ) )  
void *user_memset(void *s, char c, unsigned int n);
char *user_strcat(char* dst_, const char* src_);
int user_strcmp(const char *s1, const char *s2);
char* user_strrchr(const char* str, const unsigned char ch);
unsigned int user_strlen(const char *s);
void *user_memcpy(void *dst, const void *src, unsigned int n);
char* user_strcpy(char* dst_, const char* src_);
#endif

```



## user_syscall.c

```
#include "user_syscall.h"
#include "va_list.h"

#define MAX_ARGS            5
#define T_SYSCALL           0x80

static inline int
user_syscall(int num, ...) {
    user_va_list ap;
    user_va_start(ap, num);
    unsigned int a[MAX_ARGS];
    int i, ret;
    for (i = 0; i < MAX_ARGS; i ++) {
        a[i] = user_va_arg(ap, unsigned int);
    }
    user_va_end(ap);

    asm volatile (
        "int %1;"
        : "=a" (ret)
        : "i" (T_SYSCALL),
          "a" (num),
          "d" (a[0]),
          "c" (a[1]),
          "b" (a[2]),
          "D" (a[3]),
          "S" (a[4])
        : "cc", "memory");
    return ret;
}
int
user_sys_getpid(void) {
    return user_syscall(SYS_getpid);
}
void 
user_print_char(char c) {
    user_syscall(SYS_print_char,c);
}
void 
user_print_string(char *str) {
    user_syscall(SYS_print_string,str);
}
void 
user_print_num(int num,unsigned char base,char len,int flag) {
    user_syscall(SYS_print_num,num,base,len,flag);
}
void 
user_backtrace() {
    user_syscall(SYS_backtrace);
}
/* 从文件描述符fd中读取count个字节到buf */
int 
read(int fd, void* buf, unsigned int count) {
   return user_syscall(SYS_fdread, fd, buf, count);
}

/* 以flag方式打开文件pathname */
int open(char* pathname, unsigned char flag) {
   return user_syscall(SYS_open, pathname, flag);
}

/* 关闭文件fd */
int close(int fd) {
   return user_syscall(SYS_close, fd);
}

/* 把buf中count个字符写入文件描述符fd */
unsigned int write(int fd, const void* buf, int count) {
   return user_syscall(SYS_write, fd, buf, count);
}


/* 设置文件偏移量 */
int lseek(int fd, int offset, unsigned char whence) {
   return user_syscall(SYS_lseek, fd, offset, whence);
}

/* 删除文件pathname */
int unlink(const char* pathname) {
   return user_syscall(SYS_unlink, pathname);
}

/* 创建目录pathname */
int mkdir(const char* pathname) {
   return user_syscall(SYS_mkdir, pathname);
}

/* 删除目录pathname */
int rmdir(const char* pathname) {
   return user_syscall(SYS_rmdir, pathname);
}


/* 回归目录指针 */
void rewinddir(struct dir* dir) {
   user_syscall(SYS_rewinddir, dir);
}

/* 获取当前工作目录 */
char* getcwd(char* buf, unsigned int size) {
   return (char*)user_syscall(SYS_getcwd, buf, size);
}

/* 改变工作目录为path */
int chdir(const char* path) {
   return user_syscall(SYS_chdir, path);
}

/* 获取path属性到buf中 */
int stat(const char* path, struct stat* buf) {
   return user_syscall(SYS_stat, path, buf);
}

/* 打开目录name */
struct dir* opendir(const char* name) {
   return (struct dir*)user_syscall(SYS_opendir, name);
}

/* 关闭目录dir */
int closedir(struct dir* dir) {
   return user_syscall(SYS_closedir, dir);
}

/* 读取目录dir */
struct dir_entry* readdir(struct dir* dir) {
   return (struct dir_entry*)user_syscall(SYS_readdir, dir);
}

/* 打印全部进程 */
void ps(){
  return user_syscall(SYS_print_task);
}

/* 获取内存 */
unsigned int malloc(unsigned int bytes){
   return user_syscall(SYS_malloc,bytes);
}

/* 释放内存 */
void free(unsigned int addr,unsigned int size){
   user_syscall(SYS_free,addr,size);
}

/* 拷贝进程 */
int fork(){
   return user_syscall(SYS_fork);
}
/* 映射一段内存 */
void mmap(unsigned int va_start,unsigned int va_end,unsigned pa_start){
   return user_syscall(SYS_mmap,va_start,va_end,pa_start);
}
/* 执行程序 */
void exec(char *path,char **argv){
   return user_syscall(SYS_exec,path,argv);
}
/* 等待子进程结束 */
int wait(int* status){
   return user_syscall(SYS_wait,status);
}
/* 进程退出 */
void exit(int status){
   return user_syscall(SYS_exit,status);
}
/* 创建管道 */
void pipe(int fd[2]){
   return user_syscall(SYS_pipe,fd);
}

```



## user_syscall.h

```
#ifndef _USER_SYSCALL_H_
#define _USER_SYSCALL_H_
#include "../file/dir.h"
#include "../file/fs.h"
/* syscall number */
#define SYS_exit            1
#define SYS_fork            2
#define SYS_wait            3
#define SYS_exec            4
#define SYS_clone           5
#define SYS_yield           10
#define SYS_sleep           11
#define SYS_kill            12
#define SYS_gettime         17
#define SYS_getpid          18
#define SYS_brk             19
#define SYS_mmap            20
#define SYS_munmap          21
#define SYS_shmem           22

#define SYS_fdread            24

#define SYS_pgdir           31

#define SYS_print_char      36
#define SYS_print_string    37
#define SYS_print_num       38
#define SYS_backtrace       39
#define SYS_open  40
#define SYS_close 41
#define SYS_write 42
#define SYS_lseek 43
#define SYS_unlink 44
#define SYS_mkdir 45
#define SYS_rmdir 46
#define SYS_rewinddir 47
#define SYS_getcwd 48
#define SYS_chdir 49
#define SYS_stat 50
#define SYS_opendir 51
#define SYS_closedir 52
#define SYS_readdir 53
#define SYS_print_task 54
#define SYS_malloc 55
#define SYS_free 56
#define SYS_mmap 57
#define SYS_pipe 58
//__attribute__( (section(".user.text") ) ) 
enum std_fd{
    stdin_no, //0 标准输入
    stdout_no, //1 标准输出
    stderr_no //2 标准错误
};
/*
*__attribute__( (section(".user.text") ) )将其设置为特定的.user.text节
*方便在链接脚本中区分kernel部分和user部分
*/
//__attribute__( (section(".user.text") ) ) 
static inline int user_syscall(int num, ...);
//__attribute__( (section(".user.text") ) ) 
int user_sys_getpid(void);
//__attribute__( (section(".user.text") ) ) 
void user_print_char(char c);
//__attribute__( (section(".user.text") ) ) 
void user_print_string(char *str);
//__attribute__( (section(".user.text") ) ) 
void user_print_num(int num,unsigned char base,char len,int flag);
//__attribute__( (section(".user.text") ) ) 
void user_backtrace();
int read(int fd, void* buf, unsigned int count);
int open(char* pathname, unsigned char flag);
int close(int fd);
unsigned int write(int fd, const void* buf, int count);
int lseek(int fd, int offset, unsigned char whence) ;
int unlink(const char* pathname);
int mkdir(const char* pathname);
int rmdir(const char* pathname);
void rewinddir(struct dir* dir);
char* getcwd(char* buf, unsigned int size);
int chdir(const char* path);
int stat(const char* path, struct stat* buf) ;
struct dir* opendir(const char* name);
int closedir(struct dir* dir);
struct dir_entry* readdir(struct dir* dir);
void ps();
unsigned int malloc(unsigned int bytes);
void free(unsigned int addr,unsigned int size);
int fork();
void mmap(unsigned int va_start,unsigned int va_end,unsigned pa_start);
void exec(char *path,char **argv);
int wait(int* status);
void exit(int status);
void pipe(int fd[2]);
#endif
```



## buildin_cmd.c

```
#include "buildin_cmd.h"
#include "string.h"
#include "user_syscall.h"
#include "stdio.h"
extern char final_path[MAX_PATH_LEN];
/* 将最上层路径名称解析出来 */
static char* path_parse(char* pathname, char* name_store) {
   if (pathname[0] == '/') {   // 根目录不需要单独解析
    /* 路径中出现1个或多个连续的字符'/',将这些'/'跳过,如"///a/b" */
       while(*(++pathname) == '/');
   }

   /* 开始一般的路径解析 */
   while (*pathname != '/' && *pathname != 0) {
      *name_store++ = *pathname++;
   }

   if (pathname[0] == 0) {   // 若路径字符串为空则返回NULL
      return NULL;
   }
   return pathname; 
}
/* 将路径old_abs_path中的..和.转换为实际路径后存入new_abs_path */
static void wash_path(char* old_abs_path, char* new_abs_path) {
   char name[MAX_FILE_NAME_LEN] = {0};    
   char* sub_path = old_abs_path;
   sub_path = path_parse(sub_path, name);
   if (name[0] == 0) { // 若只键入了"/",直接将"/"存入new_abs_path后返回 
      new_abs_path[0] = '/';
      new_abs_path[1] = 0;
      return;
   }
   new_abs_path[0] = 0;	   // 避免传给new_abs_path的缓冲区不干净
   user_strcat(new_abs_path, "/");
   while (name[0]) {
      /* 如果是上一级目录“..” */
      if (!user_strcmp("..", name)) {
	 char* slash_ptr =  user_strrchr(new_abs_path, '/');
       /*如果未到new_abs_path中的顶层目录,就将最右边的'/'替换为0,
	 这样便去除了new_abs_path中最后一层路径,相当于到了上一级目录 */
	 if (slash_ptr != new_abs_path) {	// 如new_abs_path为“/a/b”,".."之后则变为“/a”
	    *slash_ptr = 0;
	 } else {	      // 如new_abs_path为"/a",".."之后则变为"/"
      /* 若new_abs_path中只有1个'/',即表示已经到了顶层目录,
	 就将下一个字符置为结束符0. */
	    *(slash_ptr + 1) = 0;
	 }
      } else if (user_strcmp(".", name)) {	  // 如果路径不是‘.’,就将name拼接到new_abs_path
	 if (user_strcmp(new_abs_path, "/")) {	  // 如果new_abs_path不是"/",就拼接一个"/",此处的判断是为了避免路径开头变成这样"//"
	    user_strcat(new_abs_path, "/");
	 }
	 user_strcat(new_abs_path, name);
      }  // 若name为当前目录".",无须处理new_abs_path

      /* 继续遍历下一层路径 */
      user_memset(name, 0, MAX_FILE_NAME_LEN);
      if (sub_path) {
	 sub_path = path_parse(sub_path, name);
      }
   }
}

/* 将path处理成不含..和.的绝对路径,存储在final_path */
void make_clear_abs_path(char* path, char* final_path) {
   char abs_path[MAX_PATH_LEN] = {0};
   /* 先判断是否输入的是绝对路径 */
   if (path[0] != '/') {      // 若输入的不是绝对路径,就拼接成绝对路径
      user_memset(abs_path, 0, MAX_PATH_LEN);
      if (getcwd(abs_path, MAX_PATH_LEN) != NULL) {
	 if (!((abs_path[0] == '/') && (abs_path[1] == 0))) {	     // 若abs_path表示的当前目录不是根目录/
	    user_strcat(abs_path, "/");
	 }
      }
   }
   user_strcat(abs_path, path);
   wash_path(abs_path, final_path);
}
/* pwd命令的内建函数 */
void buildin_pwd(int argc, char** argv __attribute__ ((unused))) {
   if (argc != 1) {
      printf("pwd: no argument support!\n");
      return;
   } else {
      if (NULL != getcwd(final_path, MAX_PATH_LEN)) {
	 printf("%s\n", final_path); 
      } else {
	 printf("pwd: get current work directory failed.\n");
      }
   }
}

/* cd命令的内建函数 */
char* buildin_cd(int argc, char** argv) {
   
   if (argc > 2) {
      printf("cd: only support 1 argument!\n");
      return NULL;
   }

   /* 若是只键入cd而无参数,直接返回到根目录. */
   if (argc == 1) {
      final_path[0] = '/';
      final_path[1] = 0;
   } else {
      make_clear_abs_path(argv[1], final_path);
   }

   if (chdir(final_path) == -1) {
      printf("cd: no such directory %s\n", final_path);
      return NULL;
   }
   return final_path;
}

/* ls命令的内建函数 */
void buildin_ls(int argc, char** argv) {
   char* pathname = NULL;
   struct stat file_stat;
   user_memset(&file_stat, 0, sizeof(struct stat));
   char long_info = 0;
   int arg_path_nr = 0;
   int arg_idx = 1;   // 跨过argv[0],argv[0]是字符串“ls”
   while (arg_idx < argc) {
      if (argv[arg_idx][0] == '-') {	  // 如果是选项,单词的首字符是-
	 if (!user_strcmp("-l", argv[arg_idx])) {         // 如果是参数-l
	    long_info = 1;
	 } else if (!user_strcmp("-h", argv[arg_idx])) {   // 参数-h
	    printf("usage: -l list all infomation about the file.\n-h for help\nlist all files in the current dirctory if no option\n"); 
	    return;
	 } else {	// 只支持-h -l两个选项
	    printf("ls: invalid option %s\nTry `ls -h' for more information.\n", argv[arg_idx]);
	    return;
	 }
      } else {	     // ls的路径参数
	 if (arg_path_nr == 0) {
	    pathname = argv[arg_idx];
	    arg_path_nr = 1;
	 } else {
	    printf("ls: only support one path\n");
	    return;
	 }
      }
      arg_idx++;
   } 
   
   if (pathname == NULL) {	 // 若只输入了ls 或 ls -l,没有输入操作路径,默认以当前路径的绝对路径为参数.
      if (NULL != getcwd(final_path, MAX_PATH_LEN)) {
	 pathname = final_path;
      } else {
	 printf("ls: getcwd for default path failed\n");
	 return;
      }
   } else {
      make_clear_abs_path(pathname, final_path);
      pathname = final_path;
   }

   if (stat(pathname, &file_stat) == -1) {
      printf("ls: cannot access %s: No such file or directory\n", pathname);
      return;
   }
   if (file_stat.st_filetype == FT_DIRECTORY) {
      struct dir* dir = opendir(pathname);
      struct dir_entry* dir_e = NULL;
      char sub_pathname[MAX_PATH_LEN] = {0};
      int pathname_len = user_strlen(pathname);
      int last_char_idx = pathname_len - 1;
      user_memcpy(sub_pathname, pathname, pathname_len);
      if (sub_pathname[last_char_idx] != '/') {
	 sub_pathname[pathname_len] = '/';
	 pathname_len++;
      }
      rewinddir(dir);
      if (long_info) {
	 char ftype;
	 printf("total: %d\n", file_stat.st_size);
	 while((dir_e = readdir(dir))) {
	    ftype = 'd';
	    if (dir_e->f_type == FT_REGULAR) {
	       ftype = '-';
	    } 
	    sub_pathname[pathname_len] = 0;
	    user_strcat(sub_pathname, dir_e->filename);
	    user_memset(&file_stat, 0, sizeof(struct stat));
	    if (stat(sub_pathname, &file_stat) == -1) {
	       printf("ls: cannot access %s: No such file or directory\n", dir_e->filename);
	       return;
	    }
	    printf("%c  %d  %d  %s\n", ftype, dir_e->i_no, file_stat.st_size, dir_e->filename);
	 }
      } else {
	 while((dir_e = readdir(dir))) {
	    printf("%s ", dir_e->filename);
	 }
	 printf("\n");
      }
      closedir(dir);
   } else {
      if (long_info) {
	 printf("-  %d  %d  %s\n", file_stat.st_ino, file_stat.st_size, pathname);
      } else {
	 printf("%s\n", pathname);  
      }
   }
}
/* ps命令内建函数 */
void buildin_ps(int argc,char **argv){
   if(argc!=1){
      printf("ps: no arguement support!\n");
      return ;
   }
   ps();
}
/* mkdir命令内建函数 */
int buildin_mkdir(int argc, char** argv) {
   int ret = -1;
   if (argc != 2) {
      printf("mkdir: only support 1 argument!\n");
   } else {
      make_clear_abs_path(argv[1], final_path);
      /* 若创建的不是根目录 */
      if (user_strcmp("/", final_path)) {
	 if (mkdir(final_path) == 0) {
	    ret = 0;
	 } else {
	    printf("mkdir: create directory %s failed.\n", argv[1]);
	 }
      }
   }
   return ret;
}

/* rmdir命令内建函数 */
int buildin_rmdir(int argc, char** argv) {
   int ret = -1;
   if (argc != 2) {
      printf("rmdir: only support 1 argument!\n");
   } else {
      make_clear_abs_path(argv[1], final_path);
   /* 若删除的不是根目录 */
      if (user_strcmp("/", final_path)) {
	 if (rmdir(final_path) == 0) {
	    ret = 0;
	 } else {
	    printf("rmdir: remove %s failed.\n", argv[1]);
	 }
      }
   }
   return ret;
}

/* rm命令内建函数 */
int buildin_rm(int argc, char** argv) {
   int ret = -1;
   if (argc != 2) {
      printf("rm: only support 1 argument!\n");
   } else {
      make_clear_abs_path(argv[1], final_path);
   /* 若删除的不是根目录 */
      if (user_strcmp("/", final_path)) {
	 if (unlink(final_path) == 0) {
	    ret = 0;
	 } else {
	    printf("rm: delete %s failed.\n", argv[1]);
	 }
	    
      }
   }
   return ret;
}
void buildin_help(int argc, char** argv){
      printf("buildin commands:\n\
       ls: show directory or file information\n\
       cd: change current work directory\n\
       mkdir: create a directory\n\
       rmdir: remove a empty directory\n\
       rm: remove a regular file\n\
       pwd: show current work directory\n\
       ps: show process information\n");
}
```



## buildin_cmd.h

```
#ifndef _BUILDIN_CMD_H_
#define _BUILDIN_CMD_H_
#include "../file/fs.h"
#define MAX_FILE_NAME_LEN 16
#define NULL ((void *)0)
#define MAX_PATH_LEN 512	    // 路径最大长度

/*
enum user_file_types {
   FT_UNKNOWN,	  // 不支持的文件类型
   FT_REGULAR,	  // 普通文件
   FT_DIRECTORY	  // 目录
};

struct stat {
   unsigned int st_ino;		 // inode编号
   unsigned int st_size;		 // 尺寸
   enum file_types st_filetype;	 // 文件类型
};*/
void make_clear_abs_path(char* path, char* final_path);
void buildin_pwd(int argc, char** argv __attribute__ ((unused))) ;
char* buildin_cd(int argc, char** argv);
void buildin_ls(int argc, char** argv);
int buildin_mkdir(int argc, char** argv);
int buildin_rmdir(int argc, char** argv);
int buildin_rm(int argc, char** argv);
void buildin_help(int argc, char** argv);
#endif
```



## shell.c

```
#include "shell.h"
#include "stdio.h"
#include "user_syscall.h"
#include "string.h"
#include "buildin_cmd.h"
#define cmd_len 128	   // 最大支持键入128个字符的命令行输入
#define MAX_ARG_NR 16	   // 加上命令名外,最多支持15个参数

#define NULL (void *)0
/* 存储输入的命令 */
//__attribute__( (section(".user.data") ) )
static char cmd_line[cmd_len] = {0};

/* 用来记录当前目录,是当前目录的缓存,每次执行cd命令时会更新此内容 */
//__attribute__( (section(".user.data") ) ) 
char cwd_cache[MAX_PATH_LEN] = {0};


char final_path[MAX_PATH_LEN] = {0};      // 用于洗路径时的缓冲
static char* argv[MAX_ARG_NR];    // argv必须为全局变量，为了以后exec的程序可访问参数
int argc = -1;
//__attribute__( (section(".user.data") ) ) 
//
/* 输出提示符 */
void print_prompt(void) {
   printf("[caoye@localhost %s]$ ",cwd_cache);
}

/* 从键盘缓冲区中最多读入count个字节到buf。*/

static void user_readline(char* buf, int count) {
   char* pos = buf;
   while (read(stdin_no, pos, 1) != -1 && (pos - buf) < count) { // 在不出错情况下,直到找到回车符才返回
      switch (*pos) {
       // 找到回车或换行符后认为键入的命令结束,直接返回 
	 case '\n':
	 case '\r':
	    *pos = 0;	   // 添加cmd_line的终止字符0
	    user_print_char('\n');
	    return;

	 case '\b':
	    if (buf[0] != '\b') {		// 阻止删除非本次输入的信息
	       --pos;	   // 退回到缓冲区cmd_line中上一个字符
	       //user_print_char('\b');
          user_backtrace();
	    }
	    break;

	 // 非控制键则输出字符 
	 default:
      if(*pos!=0){
         user_print_char(*pos);
	      pos++;
      }

      }
   }
   printf("readline: can`t find enter_key in the cmd_line, max num of char is 128\n");
}
/* 分析字符串cmd_str中以token为分隔符的单词,将各单词的指针存入argv数组 */
static int cmd_parse(char* cmd_str, char** argv, char token) {
   int arg_idx = 0;
   while(arg_idx < MAX_ARG_NR) {
      argv[arg_idx] = NULL;
      arg_idx++;
   }
   char* next = cmd_str;
   int argc = 0;
   /* 外层循环处理整个命令行 */
   while(*next) {
      /* 去除命令字或参数之间的空格 */
      while(*next == token) {
	 next++;
      }
      /* 处理最后一个参数后接空格的情况,如"ls dir2 " */
      if (*next == 0) {
	 break; 
      }
      argv[argc] = next;

     /* 内层循环处理命令行中的每个命令字及参数 */
      while (*next && *next != token) {	  // 在字符串结束前找单词分隔符
	 next++;
      }

      /* 如果未结束(是token字符),使tocken变成0 */
      if (*next) {
	 *next++ = 0;	// 将token字符替换为字符串结束符0,做为一个单词的结束,并将字符指针next指向下一个字符
      }
   
      /* 避免argv数组访问越界,参数过多则返回0 */
      if (argc > MAX_ARG_NR) {
	 return -1;
      }
      argc++;
   }
   return argc;
}


/* 简单的shell */
void my_shell(void) {
   cwd_cache[0] = '/';
   cwd_cache[1] = '\0';
   int status;
  /* unsigned int addr=malloc(500);
   printf("user malloc addr:%08x\n",addr);
   printf("before change:%08x",*(unsigned int *)addr);
   *(unsigned int *)addr=0xFFFFFFFF;
   printf("after change:%08x",*(unsigned int *)addr);
   free(addr,500);  //由于释放后，映射关系也会取消，导致缺页
   printf("after free:%08x",*(unsigned int *)addr);  */
   while (1) {
      print_prompt(); 
      user_memset(final_path, 0, MAX_PATH_LEN);
      user_memset(cmd_line, 0, cmd_len);
      user_readline(cmd_line, cmd_len);
      //printf("%s",cmd_line);
      if (cmd_line[0] == 0) {	 // 若只键入了一个回车
	        continue;
      }
      argc = -1;
      argc = cmd_parse(cmd_line, argv, ' ');
      if (argc == -1) {
         printf("num of arguments exceed %d\n", MAX_ARG_NR);
         continue;
      }
       if (!user_strcmp("ls", argv[0])) {
	      buildin_ls(argc, argv);
      } else if (!user_strcmp("cd", argv[0])) {
	 if (buildin_cd(argc, argv) != NULL) {
	    user_memset(cwd_cache, 0, MAX_PATH_LEN);
	    user_strcpy(cwd_cache, final_path);
	 }
    } else if (!user_strcmp("ps", argv[0])) {
	 buildin_ps(argc, argv);
      } else if (!user_strcmp("pwd", argv[0])) {
	 buildin_pwd(argc, argv);
      } else if (!user_strcmp("mkdir", argv[0])){
	 buildin_mkdir(argc, argv);
      } else if (!user_strcmp("rmdir", argv[0])){
	 buildin_rmdir(argc, argv);
      } else if (!user_strcmp("rm", argv[0])) {
	 buildin_rm(argc, argv);
      } else if (!user_strcmp("help", argv[0])) {
    buildin_help(argc,argv);
      } else {
         int pid = fork();
            if (pid) {	   // 父进程pid>0
               /* 下面这个while必须要加上,否则父进程一般情况下会比子进程先执行,
               因此会进行下一轮循环将findl_path清空,这样子进程将无法从final_path中获得参数*/
               int child_pid=wait(&status);
               if(child_pid==-1){
                  printf("ERROR!\n");
               }
               printf("child_pid is %d,its status is %d\n",child_pid,status);
               } else {	   // 子进程pid=0
                  make_clear_abs_path(argv[0], final_path);
                  argv[0] = final_path;
                  /* 先判断下文件是否存在 */
                  struct stat file_stat;
                  user_memset(&file_stat, 0, sizeof(struct stat));
                  if (stat(argv[0], &file_stat) == -1) {
                  printf("my_shell: cannot access %s: No such file or directory\n", argv[0]);
                  } else {
                  exec(argv[0], argv);
                  }
                  exit(-1);
               }
      }
      int arg_idx = 0;
		while(arg_idx < MAX_ARG_NR) {
			argv[arg_idx] = NULL;
			arg_idx++;
		}
   }
}

```



## shell.h

```
#ifndef _SHELL_H_
#define _SHELL_H_
//__attribute__( (section(".user.text") ) )
 void print_prompt(void);
//__attribute__( (section(".user.text") ) ) 
static void user_readline(char* buf, int count);
//__attribute__( (section(".user.text") ) ) 
void my_shell(void);
#endif
```



## user_main.c

```
#include "user_main.h"
#include "stdio.h"
#include "shell.h"
// 定义一个延时xus毫秒的延时函数
static void delay(unsigned int xus) // xus代表需要延时的微秒数
{
    unsigned int x,y;
    for(x=xus;x>0;x--)
        for(y=110;y>0;y--);
}
void test_fork(){
    unsigned int ret_pid=fork();
    printf("my pid is %d",ret_pid);
    while(1);
}
void user_main(){
    char str[50]="Hello,I'm a User Function!Nice to meet you!\n";
    //printf(str);
    //test_fork();
    my_shell();
    /*while(1)
    {
        printf("user\n");
        delay(1000000);
    }*/
    
}
```



## user_main.h

```
#ifndef _USER_MAIN_H_
#define _USER_MAIN_H_

//__attribute__( (section(".user.text") ) ) 
void user_main();

#endif
```

关于用户目录，首先要说的是这部分代码和内核代码是分开存储的，从elf文件头信息中可以看到区别。

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-R95cReiR-1609682583200)(readme.assets/image-20210103111642543.png)\]](https://img-blog.csdnimg.cn/20210103220321898.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


这部分用户代码和内核代码一样都是需要常驻内存的，其主要区别是地址空间不同还有内存页权限不同，那么说一下用户代码主要做了些啥吧。

1、提供了系统调用的接口。

2、实现了用户命令，包括内部命令和外部命令，简单讲下他们的区别吧，外部命令是说该命令是存储在文件系统的外部程序，执行该命令实际上是从文件系统上加载该程序到内存后运行的过程，也就是说外部命令会以进程的方式执行，比如ls命令，它通常的存储路径是/bin/ls。内部命令也叫内建命令，是系统本身提供的功能，它们并不以单独的程序文件存在，只是提供一些单独的功能函数。在FreeFlyOS中，为了简单，将ls、cd等命令均弄成了内部命令，外部命令主要有cat、pipe、prog等，以文件形式存在于根目录下。

3、提供了用户系统交互的shell。

大概就这么多吧，这些命令的实现还是比较简单的，搞清楚每个系统调用干了啥基本就OK了。


