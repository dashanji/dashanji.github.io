---
title: Chapter-16 file部分详解
description: Chapter 16 of FreeFlyOS
toc: true
authors:
tags:
weight: 16
categories:
series:
date: '2022-03-24'
lastmod: '2022-03-24'
draft: false
---
## bitmap.h

```
#ifndef _BITMAP_H_
#define _BITMAP_H_
/* 
** 在遍历位图时,整体上以字节为单位,细节上是以位为单位,
**     所以此处位图的指针必须是单字节 
*/
struct bitmap {
   unsigned int btmp_bytes_len;
   unsigned char* bits;
};
#endif
```



## dir.c

```
#include "dir.h"
#include "inode.h"
#include "file.h"
#include "../mem/vmm.h"
#include "../asm/asm.h"
#include "../debug/debug.h"
#define NULL (void *)0
#define BITMAP_MASK 1

struct dir *root_dir;             // 根目录
extern struct partition *cur_part;

/*
** 打开根目录 
*/
void 
open_root_dir(struct partition* part) {
   root_dir=(struct dir *)vmm_malloc(sizeof(struct dir),2);
   root_dir->inode = inode_open(part, part->sb->root_inode_no);
   root_dir->dir_pos = 0;
}

/* 在分区part上打开i结点为inode_no的目录并返回目录指针 */
struct dir* 
dir_open(struct partition* part, unsigned int inode_no) {
   struct dir* pdir = (struct dir*)vmm_malloc(sizeof(struct dir),2);
   pdir->inode = inode_open(part, inode_no);
   pdir->dir_pos = 0;
   return pdir;
}

/* 在part分区内的pdir目录内寻找名为name的文件或目录,
 * 找到后返回1并将其目录项存入dir_e,否则返回0 */
char 
search_dir_entry(struct partition* part, struct dir* pdir,
const char* name, struct dir_entry* dir_e) {
   unsigned int block_cnt = 140;	 // 12个直接块+128个一级间接块=140块

   /* 12个直接块大小+128个间接块,共560字节 */
   unsigned int* all_blocks = (unsigned int*)vmm_malloc(48 + 512,2);
   if (all_blocks == NULL) {
      printk("search_dir_entry: sys_malloc for all_blocks failed");
      return 0;
   }

   unsigned int block_idx = 0;
   while (block_idx < 12) {
      all_blocks[block_idx] = pdir->inode->i_sectors[block_idx];
      block_idx++;
   }
   block_idx = 0;

   if (pdir->inode->i_sectors[12] != 0) {	// 若含有一级间接块表
      ide_read(all_blocks + 12, pdir->inode->i_sectors[12] , 1);
   }
/* 至此,all_blocks存储的是该文件或目录的所有扇区地址 */

   /* 写目录项的时候已保证目录项不跨扇区,
    * 这样读目录项时容易处理, 只申请容纳1个扇区的内存 */
   unsigned char* buf = (unsigned char*)vmm_malloc(SECTSIZE,2);
   struct dir_entry* p_de = (struct dir_entry*)buf;	    // p_de为指向目录项的指针,值为buf起始地址
   unsigned int dir_entry_size = part->sb->dir_entry_size;
   unsigned int dir_entry_cnt = SECTSIZE / dir_entry_size;   // 1扇区内可容纳的目录项个数

   /* 开始在所有块中查找目录项 */
   while (block_idx < block_cnt) {		  
   /* 块地址为0时表示该块中无数据,继续在其它块中找 */
      if (all_blocks[block_idx] == 0) {
        block_idx++;
        continue;
      }
      ide_read(buf, all_blocks[block_idx], 1);
      
      unsigned int dir_entry_idx = 0;
      /* 遍历扇区中所有目录项 */
      while (dir_entry_idx < dir_entry_cnt) {
	   /* 若找到了,就直接复制整个目录项 */
         if (!strcmp(p_de->filename, name)) {
            memcpy(dir_e, p_de, dir_entry_size);
            vmm_free(buf,SECTSIZE);
            vmm_free(all_blocks,48 + 512);
            return 1;
         }
         dir_entry_idx++;
         p_de++;
      }
      block_idx++;
      p_de = (struct dir_entry*)buf;  // 此时p_de已经指向扇区内最后一个完整目录项了,需要恢复p_de指向为buf
      memset(buf, 0, SECTSIZE);	  // 将buf清0,下次再用
   }
	vmm_free(buf,SECTSIZE);
	vmm_free(all_blocks,48 + 512);
   return 0;
}

/* 关闭目录 */
void 
dir_close(struct dir* dir) {
/*************      根目录不能关闭     ***************
 *1 根目录自打开后就不应该关闭,否则还需要再次open_root_dir();
 *2 root_dir所在的内存是低端1M之内,并非在堆中,free会出问题 */
   if (dir == root_dir) {
   /* 不做任何处理直接返回*/
      return;
   }
   inode_close(dir->inode);
   vmm_free(dir,sizeof(struct dir));
}

/* 在内存中初始化目录项p_de */
void 
create_dir_entry(char* filename, unsigned int inode_no, 
unsigned char file_type, struct dir_entry* p_de) {
   ASSERT(strlen(filename) <=  MAX_FILE_NAME_LEN);

   /* 初始化目录项 */
   memcpy(p_de->filename, filename, strlen(filename));
   p_de->i_no = inode_no;
   p_de->f_type = file_type;
}

/* 将目录项p_de写入父目录parent_dir中,io_buf由主调函数提供 */
char sync_dir_entry(struct dir* parent_dir, struct dir_entry* p_de, void* io_buf) {
   struct inode* dir_inode = parent_dir->inode;
   unsigned int dir_size = dir_inode->i_size;
   unsigned int dir_entry_size = cur_part->sb->dir_entry_size;

   ASSERT(dir_size % dir_entry_size == 0);	 // dir_size应该是dir_entry_size的整数倍

   unsigned int dir_entrys_per_sec = (512 / dir_entry_size);       // 每扇区最大的目录项数目
   int block_lba = -1;

   /* 将该目录的所有扇区地址(12个直接块+ 128个间接块)存入all_blocks */
   unsigned char block_idx = 0;
   unsigned int all_blocks[140] = {0};	  // all_blocks保存目录所有的块

   /* 将12个直接块存入all_blocks */
   while (block_idx < 12) {
      all_blocks[block_idx] = dir_inode->i_sectors[block_idx];
      block_idx++;
   }

   struct dir_entry* dir_e = (struct dir_entry*)io_buf;	       // dir_e用来在io_buf中遍历目录项
   int block_bitmap_idx = -1;

   /* 开始遍历所有块以寻找目录项空位,若已有扇区中没有空闲位,
    * 在不超过文件大小的情况下申请新扇区来存储新目录项 */
   block_idx = 0;
   while (block_idx < 140) {  
      // 文件(包括目录)最大支持12个直接块+128个间接块＝140个块
      block_bitmap_idx = -1;
      if (all_blocks[block_idx] == 0) {   
         // 在三种情况下分配块
         block_lba = block_bitmap_alloc(cur_part);

         if (block_lba == -1) {
            printk("alloc block bitmap for sync_dir_entry failed\n");
            return 0;
	      }

         /* 每分配一个块就同步一次block_bitmap */
	      block_bitmap_idx = block_lba - cur_part->sb->data_start_lba;
	      ASSERT(block_bitmap_idx != -1);
	      bitmap_sync(cur_part, block_bitmap_idx, BLOCK_BITMAP);

	      block_bitmap_idx = -1;
	      if (block_idx < 12) {	    // 若是直接块
	         dir_inode->i_sectors[block_idx] = all_blocks[block_idx] = block_lba;
	      } 
         else if (block_idx == 12) {	  // 若是尚未分配一级间接块表(block_idx等于12表示第0个间接块地址为0)
	         dir_inode->i_sectors[12] = block_lba;       // 将上面分配的块做为一级间接块表地址
            block_lba = -1;
            block_lba = block_bitmap_alloc(cur_part);	       // 再分配一个块做为第0个间接块
            
            if (block_lba == -1) {
               block_bitmap_idx = dir_inode->i_sectors[12] - cur_part->sb->data_start_lba;
               unsigned int byte_idx = block_bitmap_idx / 8;    // 向下取整用于索引数组下标
               unsigned int bit_odd  = block_bitmap_idx % 8;    // 取余用于索引数组内的位
               cur_part->block_bitmap.bits[byte_idx]&= ~ (BITMAP_MASK << bit_odd); //将空闲位置0
               dir_inode->i_sectors[12] = 0;
               printk("alloc block bitmap for sync_dir_entry failed\n");
               return 0;
            }
         /* 每分配一个块就同步一次block_bitmap */
            block_bitmap_idx = block_lba - cur_part->sb->data_start_lba;
            ASSERT(block_bitmap_idx != -1);
            bitmap_sync(cur_part, block_bitmap_idx, BLOCK_BITMAP);

            all_blocks[12] = block_lba;
            /* 把新分配的第0个间接块地址写入一级间接块表 */
            ide_write(all_blocks + 12, dir_inode->i_sectors[12] , 1);
         } 
         else {	   
            // 若是间接块未分配
            all_blocks[block_idx] = block_lba;
            /* 把新分配的第(block_idx-12)个间接块地址写入一级间接块表 */
            ide_write(all_blocks + 12, dir_inode->i_sectors[12], 1);
         }
         /* 再将新目录项p_de写入新分配的间接块 */
         memset(io_buf, 0, 512);
         memcpy(io_buf, p_de, dir_entry_size);
         ide_write(io_buf, all_blocks[block_idx], 1);
         dir_inode->i_size += dir_entry_size;
         return 1;
      }
      /* 若第block_idx块已存在,将其读进内存,然后在该块中查找空目录项 */
      ide_read(io_buf, all_blocks[block_idx] , 1); 
      /* 在扇区内查找空目录项 */
      unsigned char dir_entry_idx = 0;
      while (dir_entry_idx < dir_entrys_per_sec) {
         if ((dir_e + dir_entry_idx)->f_type == FT_UNKNOWN) {	// FT_UNKNOWN为0,无论是初始化或是删除文件后,都会将f_type置为FT_UNKNOWN.
            memcpy(dir_e + dir_entry_idx, p_de, dir_entry_size);    
            ide_write(io_buf, all_blocks[block_idx], 1);

            dir_inode->i_size += dir_entry_size;
            return 1;
	      }
	      dir_entry_idx++;
      }
      block_idx++;
   }   
   printk("directory is full!\n");
   return 0;
}

/* 把分区part目录pdir中编号为inode_no的目录项删除 */
char delete_dir_entry(struct partition* part, struct dir* pdir, unsigned int inode_no, void* io_buf) {
   struct inode* dir_inode = pdir->inode;
   unsigned int block_idx = 0, all_blocks[140] = {0};
   unsigned int byte_idx;
   unsigned int bit_odd;
   /* 收集目录全部块地址 */
   while (block_idx < 12) {
      all_blocks[block_idx] = dir_inode->i_sectors[block_idx];
      block_idx++;
   }
   if (dir_inode->i_sectors[12]) {
      ide_read(all_blocks + 12, dir_inode->i_sectors[12],  1);
   }

   /* 目录项在存储时保证不会跨扇区 */
   unsigned int dir_entry_size = part->sb->dir_entry_size;
   unsigned int dir_entrys_per_sec = (SEC_SIZE / dir_entry_size);       // 每扇区最大的目录项数目
   struct dir_entry* dir_e = (struct dir_entry*)io_buf;   
   struct dir_entry* dir_entry_found = NULL;
   unsigned char dir_entry_idx, dir_entry_cnt;
   char is_dir_first_block = 0;     // 目录的第1个块 

   /* 遍历所有块,寻找目录项 */
   block_idx = 0;
   while (block_idx < 140) {
      is_dir_first_block = 0;
      if (all_blocks[block_idx] == 0) {
	      block_idx++;
	      continue;
      }
      dir_entry_idx = dir_entry_cnt = 0;
      memset(io_buf, 0, SEC_SIZE);
      /* 读取扇区,获得目录项 */
      ide_read(io_buf, all_blocks[block_idx],  1);

      /* 遍历所有的目录项,统计该扇区的目录项数量及是否有待删除的目录项 */
      while (dir_entry_idx < dir_entrys_per_sec) {
	      if ((dir_e + dir_entry_idx)->f_type != FT_UNKNOWN) {
	         if (!strcmp((dir_e + dir_entry_idx)->filename, ".")) { 
	            is_dir_first_block = 1;
	         } 
            else if (strcmp((dir_e + dir_entry_idx)->filename, ".") 
            && strcmp((dir_e + dir_entry_idx)->filename, "..")) {
	            dir_entry_cnt++;     // 统计此扇区内的目录项个数,用来判断删除目录项后是否回收该扇区
	            if ((dir_e + dir_entry_idx)->i_no == inode_no) {	  // 如果找到此i结点,就将其记录在dir_entry_found
                  ASSERT(dir_entry_found == NULL);  // 确保目录中只有一个编号为inode_no的inode,找到一次后dir_entry_found就不再是NULL
                  dir_entry_found = dir_e + dir_entry_idx;
		  /* 找到后也继续遍历,统计总共的目录项数 */
	            }
	         }
	      }
	      dir_entry_idx++;
      } 

      /* 若此扇区未找到该目录项,继续在下个扇区中找 */
      if (dir_entry_found == NULL) {
         block_idx++;
         continue;
      }

      /* 在此扇区中找到目录项后,清除该目录项并判断是否回收扇区,随后退出循环直接返回 */
      ASSERT(dir_entry_cnt >= 1);
      /* 除目录第1个扇区外,若该扇区上只有该目录项自己,则将整个扇区回收 */
      if (dir_entry_cnt == 1 && !is_dir_first_block) {
         /* a 在块位图中回收该块 */
         unsigned int block_bitmap_idx = all_blocks[block_idx] - part->sb->data_start_lba;
         byte_idx = block_bitmap_idx / 8;
         bit_odd  = block_bitmap_idx % 8;
         part->block_bitmap.bits[byte_idx] &= ~(1 << bit_odd);
         bitmap_sync(cur_part, block_bitmap_idx, BLOCK_BITMAP);

         /* b 将块地址从数组i_sectors或索引表中去掉 */
         if (block_idx < 12) {
            dir_inode->i_sectors[block_idx] = 0;
         } else {    // 在一级间接索引表中擦除该间接块地址
         /*先判断一级间接索引表中间接块的数量,如果仅有这1个间接块,连同间接索引表所在的块一同回收 */
            unsigned int indirect_blocks = 0;
            unsigned int indirect_block_idx = 12;
            while (indirect_block_idx < 140) {
               if (all_blocks[indirect_block_idx] != 0) {
                  indirect_blocks++;
               }
            }
            ASSERT(indirect_blocks >= 1);  // 包括当前间接块

            if (indirect_blocks > 1) {	  // 间接索引表中还包括其它间接块,仅在索引表中擦除当前这个间接块地址
               all_blocks[block_idx] = 0; 
               ide_write(all_blocks + 12, dir_inode->i_sectors[12], 1); 
            } else {	// 间接索引表中就当前这1个间接块,直接把间接索引表所在的块回收,然后擦除间接索引表块地址
               /* 回收间接索引表所在的块 */
               block_bitmap_idx = dir_inode->i_sectors[12] - part->sb->data_start_lba;
               byte_idx = block_bitmap_idx / 8;
               bit_odd  = block_bitmap_idx % 8;
               part->block_bitmap.bits[byte_idx] &= ~(1 << bit_odd);
               bitmap_sync(cur_part, block_bitmap_idx, BLOCK_BITMAP);
               
               /* 将间接索引表地址清0 */
               dir_inode->i_sectors[12] = 0;
            }
         }
      } else { // 仅将该目录项清空
            memset(dir_entry_found, 0, dir_entry_size);
            ide_write(io_buf,  all_blocks[block_idx], 1);
         }

      /* 更新i结点信息并同步到硬盘 */
      ASSERT(dir_inode->i_size >= dir_entry_size);
      dir_inode->i_size -= dir_entry_size;
      memset(io_buf, 0, SEC_SIZE * 2);
      inode_sync(part, dir_inode, io_buf);

      return 1;
   }
   /* 所有块中未找到则返回false,若出现这种情况应该是serarch_file出错了 */
   return 0;
}
/* 读取目录,成功返回1个目录项,失败返回NULL */
struct dir_entry* dir_read(struct dir* dir) {
   struct dir_entry* dir_e = (struct dir_entry*)dir->dir_buf;
   struct inode* dir_inode = dir->inode; 
   unsigned int all_blocks[140] = {0}, block_cnt = 12;
   unsigned int block_idx = 0, dir_entry_idx = 0;
   while (block_idx < 12) {
      all_blocks[block_idx] = dir_inode->i_sectors[block_idx];
      block_idx++;
   }
   if (dir_inode->i_sectors[12] != 0) {	     // 若含有一级间接块表
      ide_read(all_blocks + 12, dir_inode->i_sectors[12],  1);
      block_cnt = 140;
   }
   block_idx = 0;

   unsigned int cur_dir_entry_pos = 0;	  // 当前目录项的偏移,此项用来判断是否是之前已经返回过的目录项
   unsigned int dir_entry_size = cur_part->sb->dir_entry_size;
   unsigned int dir_entrys_per_sec = SEC_SIZE / dir_entry_size;	 // 1扇区内可容纳的目录项个数
   /* 因为此目录内可能删除了某些文件或子目录,所以要遍历所有块 */
   while (block_idx < block_cnt) {
      if (dir->dir_pos >= dir_inode->i_size) {
	      return NULL;
      }
      if (all_blocks[block_idx] == 0) {     // 如果此块地址为0,即空块,继续读出下一块
	      block_idx++;
	      continue;
      }
      memset(dir_e, 0, SEC_SIZE);
      ide_read(dir_e, all_blocks[block_idx],  1);
      dir_entry_idx = 0;
      /* 遍历扇区内所有目录项 */
      while (dir_entry_idx < dir_entrys_per_sec) {
	      if ((dir_e + dir_entry_idx)->f_type) {	 // 如果f_type不等于0,即不等于FT_UNKNOWN
	      /* 判断是不是最新的目录项,避免返回曾经已经返回过的目录项 */
	         if (cur_dir_entry_pos < dir->dir_pos) {
	            cur_dir_entry_pos += dir_entry_size;
	            dir_entry_idx++;
	            continue;
	         }
	         ASSERT(cur_dir_entry_pos == dir->dir_pos);
            dir->dir_pos += dir_entry_size;	      // 更新为新位置,即下一个返回的目录项地址
            return dir_e + dir_entry_idx; 
	      }
	      dir_entry_idx++;
      }
      block_idx++;
   }
   return NULL;
}
/* 判断目录是否为空 */
char dir_is_empty(struct dir* dir) {
   struct inode* dir_inode = dir->inode;
   /* 若目录下只有.和..这两个目录项则目录为空 */
   return (dir_inode->i_size == cur_part->sb->dir_entry_size * 2);
}

/* 在父目录parent_dir中删除child_dir */
int dir_remove(struct dir* parent_dir, struct dir* child_dir) {
   struct inode* child_dir_inode  = child_dir->inode;
   /* 空目录只在inode->i_sectors[0]中有扇区,其它扇区都应该为空 */
   int block_idx = 1;
   while (block_idx < 13) {
      ASSERT(child_dir_inode->i_sectors[block_idx] == 0);
      block_idx++;
   }
   void* io_buf = vmm_malloc(SEC_SIZE * 2,2);
   if (io_buf == NULL) {
      printk("dir_remove: malloc for io_buf failed\n");
      return -1;
   }

   /* 在父目录parent_dir中删除子目录child_dir对应的目录项 */
   delete_dir_entry(cur_part, parent_dir, child_dir_inode->i_num, io_buf);

   /* 回收inode中i_secotrs中所占用的扇区,并同步inode_bitmap和block_bitmap */
   inode_release(cur_part, child_dir_inode->i_num);
   vmm_free(io_buf,SEC_SIZE * 2);
   return 0;
}

```



## dir.h

```
#ifndef _DIR_H_
#define _DIR_H_
#include "fs.h"
#include "ide-dev.h"
#define MAX_FILE_NAME_LEN 16

/* 目录结构 */
struct dir
{
    struct inode *inode;
    unsigned int dir_pos;  //记录在目录内的偏移
    unsigned char dir_buf[512];  //目录的数据缓存
};
/* 目录项结构 */
struct  dir_entry
{
    char filename[MAX_FILE_NAME_LEN]; //普通文件或目录名称
    unsigned i_no;  //普通文件或目录对应的inode结点号
    enum file_types f_type; //文件类型
};
void open_root_dir(struct partition* part);
struct dir* dir_open(struct partition* part, unsigned int inode_no);
char search_dir_entry(struct partition* part, struct dir* pdir, const char* name, struct dir_entry* dir_e);
void dir_close(struct dir* dir);
void create_dir_entry(char* filename, unsigned int inode_no, unsigned char file_type, struct dir_entry* p_de);
struct dir_entry* dir_read(struct dir* dir);
#endif
```



## file.c

```
#include "file.h"
#include "fs.h"
#include "inode.h"
#include "../debug/debug.h"
#include "../vga/vga.h"
#include "../task/task.h"
#define NULL (void *)0
#define BITMAP_MASK 1
/* 文件表 */
struct file file_table[MAX_FILE_OPEN];
extern struct task_struct *current;
extern struct partition *cur_part;
/* 从文件表file_table中获取一个空闲位,成功返回下标，失败返回-1 */
int get_free_slot_in_global(void){
   unsigned int fd_idx=3;
   while(fd_idx<MAX_FILE_OPEN){
      if(file_table[fd_idx].fd_inode==NULL)
         break;
      fd_idx++;
   }
   if(fd_idx==MAX_FILE_OPEN){
      return -1;
   }
   return fd_idx;
}
/* 将全局描述符下标安装到进程或线程自己的文件描述符数组fd_table中，成功返回下标，失败返回-1 */
int task_fd_install(int global_fd_idx){
   char local_fd_idx=3; 
   while(local_fd_idx<MAX_FILE){
      if(current->fd_table[local_fd_idx]==-1){
         current->fd_table[local_fd_idx]=global_fd_idx;
         break;
      }
      local_fd_idx++;
   }
   if(local_fd_idx==MAX_FILE){
     return -1;
   }
   return local_fd_idx;
}
/* 分配一个i节点，返回i节点号 */
int inode_bitmap_alloc(struct partition *part){
   unsigned int idx_byte=0;//记录空闲位所在字节
    /* 先逐字节比较,蛮力法 */
   while (( 0xff == part->inode_bitmap.bits[idx_byte]) && (idx_byte < part->inode_bitmap.btmp_bytes_len)) {
   /* 1表示该位已分配,所以若为0xff,则表示该字节内已无空闲位,向下一字节继续找 */
      idx_byte++;
   }
   if (idx_byte == part->inode_bitmap.btmp_bytes_len) {  // 若该内存池找不到可用空间		
      return -1;
   } 

   /* 若在位图数组范围内的某字节内找到了空闲位，
  * 在该字节内逐位比对,返回空闲位的索引。*/
   int idx_bit = 0;
 /* 和btmp->bits[idx_byte]这个字节逐位对比 */
   while ((unsigned char)(BITMAP_MASK << idx_bit) & part->inode_bitmap.bits[idx_byte]) { 
	   idx_bit++;
   }
   int bit_idx_start = idx_byte * 8 + idx_bit;    // 空闲位在位图内的下标
   part->inode_bitmap.bits[idx_byte]|= (BITMAP_MASK << idx_bit); //将空闲位置1
   return bit_idx_start;
}

/* 分配1个扇区，返回扇区地址 */
int block_bitmap_alloc(struct partition *part){
   unsigned int idx_byte=0;//记录空闲位所在字节
   /* 先逐字节比较,蛮力法 */
   while (( 0xff == part->block_bitmap.bits[idx_byte]) && (idx_byte < part->block_bitmap.btmp_bytes_len)) {
   /* 1表示该位已分配,所以若为0xff,则表示该字节内已无空闲位,向下一字节继续找 */
      idx_byte++;
   }
   if (idx_byte == part->block_bitmap.btmp_bytes_len) {  // 若该内存池找不到可用空间		
      return -1;
   } 

   /* 若在位图数组范围内的某字节内找到了空闲位，
   * 在该字节内逐位比对,返回空闲位的索引。*/
   int idx_bit = 0;
   /* 和btmp->bits[idx_byte]这个字节逐位对比 */
   while ((unsigned char)(BITMAP_MASK << idx_bit) & part->block_bitmap.bits[idx_byte]) { 
	   idx_bit++;
   }
   int bit_idx_start = idx_byte * 8 + idx_bit;    // 空闲位在位图内的下标
   part->block_bitmap.bits[idx_byte]|= (BITMAP_MASK << idx_bit); //将空闲位置1
   return part->sb->data_start_lba+bit_idx_start;
}
/* 将内存中bitmap第bit_idx位所在的512字节同步到硬盘 */
void bitmap_sync(struct partition *part,unsigned int bit_idx,unsigned char btmp){
   //i节点索引相对于位图的扇区偏移量
   unsigned int off_sec=bit_idx/(512*8);
   //i节点索引相对于位图的字节偏移量
   unsigned int off_size=off_sec * SECTSIZE;
   unsigned int sec_lba;
   unsigned char *bitmap_off;

   /* 需要被同步到硬盘的位图只有inode_bitmap和block_bitmap */
   switch(btmp){
      case INODE_BITMAP:
         sec_lba=part->sb->inode_bitmap_lba+off_sec;
         bitmap_off= part->inode_bitmap.bits + off_size;
         break;
      case BLOCK_BITMAP:
         sec_lba=part->sb->block_bitmap_lba+off_sec;
         bitmap_off= part->block_bitmap.bits + off_size;
         break;
   }
   ide_write(bitmap_off,sec_lba,1);
}
/* 创建文件,若成功则返回文件描述符,否则返回-1 */
int file_create(struct dir* parent_dir, char* filename, unsigned char flag) {
   /* 后续操作的公共缓冲区 */
   void* io_buf = vmm_malloc(VMM_PAGE_SIZE,2);
   if (io_buf == NULL) {
      printk("in file_creat: sys_malloc for io_buf failed\n");
      return -1;
   }

   unsigned char rollback_step = 0;	       // 用于操作失败时回滚各资源状态

   /* 为新文件分配inode */
   int inode_no = inode_bitmap_alloc(cur_part); 
   if (inode_no == -1) {
      printk("in file_creat: allocate inode failed\n");
      return -1;
   }

/* 此inode要从堆中申请内存,不可生成局部变量(函数退出时会释放)
 * 因为file_table数组中的文件描述符的inode指针要指向它.*/
   struct inode* new_file_inode = (struct inode*)vmm_malloc(sizeof(struct inode),2); 
   if (new_file_inode == NULL) {
      printk("file_create: sys_malloc for inode failded\n");
      rollback_step = 1;
      goto rollback;
   }
   inode_init(inode_no, new_file_inode);	    // 初始化i结点

   /* 返回的是file_table数组的下标 */
   int fd_idx = get_free_slot_in_global();
   if (fd_idx == -1) {
      printk("exceed max open files\n");
      rollback_step = 2;
      goto rollback;
   }

   file_table[fd_idx].fd_inode = new_file_inode;
   file_table[fd_idx].fd_pos = 0;
   file_table[fd_idx].fd_flag = flag;
   file_table[fd_idx].fd_inode->write_lock = 0;

   struct dir_entry new_dir_entry;
   memset(&new_dir_entry, 0, sizeof(struct dir_entry));

   create_dir_entry(filename, inode_no, FT_REGULAR, &new_dir_entry);	// create_dir_entry只是内存操作不出意外,不会返回失败

/* 同步内存数据到硬盘 */
   /* a 在目录parent_dir下安装目录项new_dir_entry, 写入硬盘后返回true,否则false */
   if (!sync_dir_entry(parent_dir, &new_dir_entry, io_buf)) {
      printk("sync dir_entry to disk failed\n");
      rollback_step = 3;
      goto rollback;
   }

   memset(io_buf, 0, 1024);
   /* b 将父目录i结点的内容同步到硬盘 */
   inode_sync(cur_part, parent_dir->inode, io_buf);

   memset(io_buf, 0, 1024);
   /* c 将新创建文件的i结点内容同步到硬盘 */
   inode_sync(cur_part, new_file_inode, io_buf);

   /* d 将inode_bitmap位图同步到硬盘 */
   bitmap_sync(cur_part, inode_no, INODE_BITMAP);

   /* e 将创建的文件i结点添加到open_inodes链表 */
   //list_push(&cur_part->open_inodes, &new_file_inode->inode_tag);
   //new_file_inode->i_open_cnts = 1;

   vmm_free(io_buf,1024);
   return task_fd_install(fd_idx);
   unsigned int byte_idx;
   unsigned int bit_odd;
   /*创建文件需要创建相关的多个资源,若某步失败则会执行到下面的回滚步骤 */
rollback:
   switch (rollback_step) {
      case 3:
	   /* 失败时,将file_table中的相应位清空 */
	      memset(&file_table[fd_idx], 0, sizeof(struct file)); 
      case 2:
	      vmm_free(new_file_inode,sizeof(struct inode));
      case 1:
	      /* 如果新文件的i结点创建失败,之前位图中分配的inode_no也要恢复 */
         byte_idx = inode_no / 8;    // 向下取整用于索引数组下标
         bit_odd  = inode_no % 8;    // 取余用于索引数组内的位
         cur_part->inode_bitmap.bits[byte_idx] &= ~(BITMAP_MASK << 0);
	      break;
   }
   vmm_free(io_buf,1024);
   return -1;
}
/* 打开编号为inode_no的inode对应的文件,若成功则返回文件描述符,否则返回-1 */
int file_open(unsigned int inode_no, unsigned char flag) {
   int fd_idx = get_free_slot_in_global();
   if (fd_idx == -1) {
      printk("exceed max open files\n");
      return -1;
   }
   //printk("fd_idx:%08d\n",inode_no);
   file_table[fd_idx].fd_inode = inode_open(cur_part, inode_no);
   //printk("file_table[fd_idx].fd_inode->inum:%08x",file_table[fd_idx].fd_inode->i_num);
   file_table[fd_idx].fd_pos = 0;	     // 每次打开文件,要将fd_pos还原为0,即让文件内的指针指向开头
   file_table[fd_idx].fd_flag = flag;
   char* write_lock = &file_table[fd_idx].fd_inode->write_lock; 

   if (flag & O_WRONLY || flag & O_RDWR) {	// 只要是关于写文件,判断是否有其它进程正写此文件
						// 若是读文件,不考虑write_deny
      /* 以下进入临界区前先关中断 */
      enum intr_status flag;
      local_intr_save(flag);
      if (!(*write_lock)) {    // 若当前没有其它进程写该文件,将其占用.
         *write_lock = 1;   // 置为true,避免多个进程同时写此文件
         local_intr_restore(flag);  // 恢复中断
      } 
      else {		// 直接失败返回
         local_intr_restore(flag);  // 恢复中断
         printk("file can`t be write now, try again later\n");
         return -1;
      }
      
   }  // 若是读文件或创建文件,不用理会write_deny,保持默认
   return task_fd_install(fd_idx);
}

/* 关闭文件 */
int file_close(struct file* file) {
   if (file == NULL) {
      return -1;
   }
   file->fd_inode->write_lock = 0;
   inode_close(file->fd_inode);
   file->fd_inode = NULL;   // 使文件结构可用
   return 0;
}

/* 把buf中的count个字节写入file,成功则返回写入的字节数,失败则返回-1 */
int file_write(struct file* file, const void* buf, unsigned int count) {
   if ((file->fd_inode->i_size + count) > (SEC_SIZE * 140))	{   // 文件目前最大只支持512*140=71680字节
      printk("exceed max file_size 71680 bytes, write file failed\n");
      return -1;
   }
   unsigned char* io_buf = (unsigned char *)vmm_malloc(SEC_SIZE,2);
   if (io_buf == NULL) {
      printk("file_write: sys_malloc for io_buf failed\n");
      return -1;
   }
   unsigned int* all_blocks =(unsigned int *)vmm_malloc(SEC_SIZE + 48,2);	  // 用来记录文件所有的块地址
   if (all_blocks == NULL) {
      printk("file_write: sys_malloc for all_blocks failed\n");
      return -1;
   }

   const unsigned char* src = buf;	    // 用src指向buf中待写入的数据 
   unsigned int bytes_written = 0;	    // 用来记录已写入数据大小
   unsigned int size_left = count;	    // 用来记录未写入数据大小
   int block_lba = -1;	    // 块地址
   unsigned int block_bitmap_idx = 0;   // 用来记录block对应于block_bitmap中的索引,做为参数传给bitmap_sync
   unsigned int sec_idx;	      // 用来索引扇区
   unsigned int sec_lba;	      // 扇区地址
   unsigned int sec_off_bytes;    // 扇区内字节偏移量
   unsigned int sec_left_bytes;   // 扇区内剩余字节量
   unsigned int chunk_size;	      // 每次写入硬盘的数据块大小
   int indirect_block_table;      // 用来获取一级间接表地址
   unsigned int block_idx;		      // 块索引

   /* 判断文件是否是第一次写,如果是,先为其分配一个块 */
   if (file->fd_inode->i_sectors[0] == 0) {
      block_lba = block_bitmap_alloc(cur_part);
      if (block_lba == -1) {
         printk("file_write: block_bitmap_alloc failed\n");
         return -1;
      }
      file->fd_inode->i_sectors[0] = block_lba;

      /* 每分配一个块就将位图同步到硬盘 */
      block_bitmap_idx = block_lba - cur_part->sb->data_start_lba;
      ASSERT(block_bitmap_idx != 0);
      bitmap_sync(cur_part, block_bitmap_idx, BLOCK_BITMAP);
   }

   /* 写入count个字节前,该文件已经占用的块数 */
   unsigned int file_has_used_blocks = file->fd_inode->i_size / SEC_SIZE + 1;

   /* 存储count字节后该文件将占用的块数 */
   unsigned int file_will_use_blocks = (file->fd_inode->i_size + count) / SEC_SIZE + 1;
   ASSERT(file_will_use_blocks <= 140);

   /* 通过此增量判断是否需要分配扇区,如增量为0,表示原扇区够用 */
   unsigned int add_blocks = file_will_use_blocks - file_has_used_blocks;

/* 开始将文件所有块地址收集到all_blocks,(系统中块大小等于扇区大小)
 * 后面都统一在all_blocks中获取写入扇区地址 */
   if (add_blocks == 0) { 
   /* 在同一扇区内写入数据,不涉及到分配新扇区 */
      if (file_has_used_blocks <= 12 ) {	// 文件数据量将在12块之内
	      block_idx = file_has_used_blocks - 1;  // 指向最后一个已有数据的扇区
	      all_blocks[block_idx] = file->fd_inode->i_sectors[block_idx];
      } else { 
         /* 未写入新数据之前已经占用了间接块,需要将间接块地址读进来 */
	      ASSERT(file->fd_inode->i_sectors[12] != 0);
         indirect_block_table = file->fd_inode->i_sectors[12];
	      ide_read(all_blocks + 12, indirect_block_table, 1);
      }
   } else {
   /* 若有增量,便涉及到分配新扇区及是否分配一级间接块表,下面要分三种情况处理 */
   /* 第一种情况:12个直接块够用*/
      if (file_will_use_blocks <= 12 ) {
         /* 先将有剩余空间的可继续用的扇区地址写入all_blocks */
         block_idx = file_has_used_blocks - 1;
         ASSERT(file->fd_inode->i_sectors[block_idx] != 0);
         all_blocks[block_idx] = file->fd_inode->i_sectors[block_idx];

         /* 再将未来要用的扇区分配好后写入all_blocks */
         block_idx = file_has_used_blocks;      // 指向第一个要分配的新扇区
         while (block_idx < file_will_use_blocks) {
            block_lba = block_bitmap_alloc(cur_part);
            if (block_lba == -1) {
               printk("file_write: block_bitmap_alloc for situation 1 failed\n");
               return -1;
            }

            /* 写文件时,不应该存在块未使用但已经分配扇区的情况,当文件删除时,就会把块地址清0 */
            ASSERT(file->fd_inode->i_sectors[block_idx] == 0);     // 确保尚未分配扇区地址
            file->fd_inode->i_sectors[block_idx] = all_blocks[block_idx] = block_lba;

            /* 每分配一个块就将位图同步到硬盘 */
            block_bitmap_idx = block_lba - cur_part->sb->data_start_lba;
            bitmap_sync(cur_part, block_bitmap_idx, BLOCK_BITMAP);

            block_idx++;   // 下一个分配的新扇区
	      }
      } else if (file_has_used_blocks <= 12 && file_will_use_blocks > 12) { 
         /* 第二种情况: 旧数据在12个直接块内,新数据将使用间接块*/

         /* 先将有剩余空间的可继续用的扇区地址收集到all_blocks */
         block_idx = file_has_used_blocks - 1;      // 指向旧数据所在的最后一个扇区
         all_blocks[block_idx] = file->fd_inode->i_sectors[block_idx];

         /* 创建一级间接块表 */
         block_lba = block_bitmap_alloc(cur_part);
         if (block_lba == -1) {
            printk("file_write: block_bitmap_alloc for situation 2 failed\n");
            return -1;
         }

         ASSERT(file->fd_inode->i_sectors[12] == 0);  // 确保一级间接块表未分配
         /* 分配一级间接块索引表 */
         indirect_block_table = file->fd_inode->i_sectors[12] = block_lba;

         block_idx = file_has_used_blocks;	// 第一个未使用的块,即本文件最后一个已经使用的直接块的下一块
         while (block_idx < file_will_use_blocks) {
            block_lba = block_bitmap_alloc(cur_part);
            if (block_lba == -1) {
               printk("file_write: block_bitmap_alloc for situation 2 failed\n");
               return -1;
            }

            if (block_idx < 12) {      // 新创建的0~11块直接存入all_blocks数组
               ASSERT(file->fd_inode->i_sectors[block_idx] == 0);      // 确保尚未分配扇区地址
               file->fd_inode->i_sectors[block_idx] = all_blocks[block_idx] = block_lba;
            } else {     // 间接块只写入到all_block数组中,待全部分配完成后一次性同步到硬盘
               all_blocks[block_idx] = block_lba;
            }

            /* 每分配一个块就将位图同步到硬盘 */
            block_bitmap_idx = block_lba - cur_part->sb->data_start_lba;
            bitmap_sync(cur_part, block_bitmap_idx, BLOCK_BITMAP);

            block_idx++;   // 下一个新扇区
         }
         ide_write(all_blocks + 12, indirect_block_table, 1);      // 同步一级间接块表到硬盘
      } else if (file_has_used_blocks > 12) {
         /* 第三种情况:新数据占据间接块*/
         ASSERT(file->fd_inode->i_sectors[12] != 0); // 已经具备了一级间接块表
         indirect_block_table = file->fd_inode->i_sectors[12];	 // 获取一级间接表地址

         /* 已使用的间接块也将被读入all_blocks,无须单独收录 */
         ide_read(all_blocks + 12, indirect_block_table, 1); // 获取所有间接块地址

         block_idx = file_has_used_blocks;	  // 第一个未使用的间接块,即已经使用的间接块的下一块
         while (block_idx < file_will_use_blocks) {
            block_lba = block_bitmap_alloc(cur_part);
            if (block_lba == -1) {
               printk("file_write: block_bitmap_alloc for situation 3 failed\n");
               return -1;
            }
            all_blocks[block_idx++] = block_lba;

            /* 每分配一个块就将位图同步到硬盘 */
            block_bitmap_idx = block_lba - cur_part->sb->data_start_lba;
            bitmap_sync(cur_part, block_bitmap_idx, BLOCK_BITMAP);
         }
         ide_write(all_blocks + 12, indirect_block_table , 1);   // 同步一级间接块表到硬盘
      } 
   }

   char first_write_block = 1;      // 含有剩余空间的扇区标识
   /* 块地址已经收集到all_blocks中,下面开始写数据 */
   file->fd_pos = file->fd_inode->i_size - 1;   // 置fd_pos为文件大小-1,下面在写数据时随时更新
   while (bytes_written < count) {      // 直到写完所有数据
      memset(io_buf, 0, SEC_SIZE);
      sec_idx = file->fd_inode->i_size / SEC_SIZE;
      sec_lba = all_blocks[sec_idx];
      sec_off_bytes = file->fd_inode->i_size % SEC_SIZE;
      sec_left_bytes = SEC_SIZE - sec_off_bytes;

      /* 判断此次写入硬盘的数据大小 */
      chunk_size = size_left < sec_left_bytes ? size_left : sec_left_bytes;
      if (first_write_block) {
         ide_read(io_buf, sec_lba, 1);
         first_write_block = 0;
      }
      memcpy(io_buf + sec_off_bytes, src, chunk_size);
      ide_write(io_buf, sec_lba, 1);
     // printk("file write at lba 0x%08x\n", sec_lba);    //调试,完成后去掉

      src += chunk_size;   // 将指针推移到下个新数据
      file->fd_inode->i_size += chunk_size;  // 更新文件大小
      file->fd_pos += chunk_size;   
      bytes_written += chunk_size;
      size_left -= chunk_size;
   }
   inode_sync(cur_part, file->fd_inode, io_buf);
   vmm_free(all_blocks,SEC_SIZE + 48);
   vmm_free(io_buf,SEC_SIZE);
   return bytes_written;
}
/* 从文件file中读取count个字节写入buf, 返回读出的字节数,若到文件尾则返回-1 */
int file_read(struct file* file, void* buf, unsigned int count) {
   unsigned char* buf_dst = (unsigned char*)buf;
   unsigned int size = count, size_left = size;
   //printk("file pos %02d\n",file->fd_pos);
   //printk("file size %02d\n",file->fd_inode->i_size);
   /* 若要读取的字节数超过了文件可读的剩余量, 就用剩余量做为待读取的字节数 */
   if ((file->fd_pos + count) > file->fd_inode->i_size)	{
      size = file->fd_inode->i_size - file->fd_pos;
      size_left = size;
      if (size == 0) {	   // 若到文件尾则返回-1
	      return -1;
      }
   }
   //printk("file read size %02d\n",size);
   unsigned char* io_buf = (unsigned char *)vmm_malloc(SEC_SIZE,2);
   if (io_buf == NULL) {
      printk("file_read: sys_malloc for io_buf failed\n");
   }
   unsigned int* all_blocks = (unsigned int *)vmm_malloc(SEC_SIZE + 48,2);	  // 用来记录文件所有的块地址
   if (all_blocks == NULL) {
      printk("file_read: sys_malloc for all_blocks failed\n");
      return -1;
   }

   unsigned int block_read_start_idx = file->fd_pos / SEC_SIZE;		       // 数据所在块的起始地址
   unsigned int block_read_end_idx = (file->fd_pos + size) / SEC_SIZE;	       // 数据所在块的终止地址
   unsigned int read_blocks = block_read_start_idx - block_read_end_idx;	       // 如增量为0,表示数据在同一扇区
   ASSERT(block_read_start_idx < 139 && block_read_end_idx < 139);

   int indirect_block_table;       // 用来获取一级间接表地址
   unsigned int block_idx;		       // 获取待读的块地址 

/* 以下开始构建all_blocks块地址数组,专门存储用到的块地址(本程序中块大小同扇区大小) */
   if (read_blocks == 0) {       // 在同一扇区内读数据,不涉及到跨扇区读取
      ASSERT(block_read_end_idx == block_read_start_idx);
      if (block_read_end_idx < 12 ) {	   // 待读的数据在12个直接块之内
         block_idx = block_read_end_idx;
         all_blocks[block_idx] = file->fd_inode->i_sectors[block_idx];
      } else {		// 若用到了一级间接块表,需要将表中间接块读进来
         indirect_block_table = file->fd_inode->i_sectors[12];
         ide_read(all_blocks + 12, indirect_block_table, 1);
      }
   } else {      // 若要读多个块
      /* 第一种情况: 起始块和终止块属于直接块*/
      if (block_read_end_idx < 12 ) {	  // 数据结束所在的块属于直接块
	      block_idx = block_read_start_idx; 
	      while (block_idx <= block_read_end_idx) {
	         all_blocks[block_idx] = file->fd_inode->i_sectors[block_idx]; 
	         block_idx++;
	      }
      } else if (block_read_start_idx < 12 && block_read_end_idx >= 12) {
      /* 第二种情况: 待读入的数据跨越直接块和间接块两类*/
      /* 先将直接块地址写入all_blocks */
	      block_idx = block_read_start_idx;
	      while (block_idx < 12) {
	         all_blocks[block_idx] = file->fd_inode->i_sectors[block_idx];
	         block_idx++;
	      }
	      ASSERT(file->fd_inode->i_sectors[12] != 0);	    // 确保已经分配了一级间接块表

         /* 再将间接块地址写入all_blocks */
	      indirect_block_table = file->fd_inode->i_sectors[12];
	      ide_read(all_blocks + 12, indirect_block_table,  1);	      // 将一级间接块表读进来写入到第13个块的位置之后
      } else {	
         /* 第三种情况: 数据在间接块中*/
	      ASSERT(file->fd_inode->i_sectors[12] != 0);	    // 确保已经分配了一级间接块表
	      indirect_block_table = file->fd_inode->i_sectors[12];	      // 获取一级间接表地址
	      ide_read(all_blocks + 12, indirect_block_table,  1);	      // 将一级间接块表读进来写入到第13个块的位置之后
      } 
   }

   /* 用到的块地址已经收集到all_blocks中,下面开始读数据 */
   unsigned int sec_idx, sec_lba, sec_off_bytes, sec_left_bytes, chunk_size;
   unsigned int bytes_read = 0;
   while (bytes_read < size) {	      // 直到读完为止
      sec_idx = file->fd_pos / SEC_SIZE;
      sec_lba = all_blocks[sec_idx];
      sec_off_bytes = file->fd_pos % SEC_SIZE;
      sec_left_bytes = SEC_SIZE - sec_off_bytes;
      chunk_size = size_left < sec_left_bytes ? size_left : sec_left_bytes;	     // 待读入的数据大小

      memset(io_buf, 0, SEC_SIZE);
      ide_read(io_buf, sec_lba, 1);
      memcpy(buf_dst, io_buf + sec_off_bytes, chunk_size);

      buf_dst += chunk_size;
      file->fd_pos += chunk_size;
      bytes_read += chunk_size;
      size_left -= chunk_size;
   }
   vmm_free(all_blocks,SEC_SIZE + 48);
   vmm_free(io_buf,SEC_SIZE);
   return bytes_read;
}

```



## file.h

```
#ifndef _FILE_H_
#define _FILE_H_
#define MAX_FILE_OPEN 32 //可打开的最大文件数
#include "dir.h"
#include "ide-dev.h"
/* 文件结构 */
struct file{
    unsigned int fd_pos; 
    unsigned int fd_flag;
    struct inode *fd_inode;
};
/* 标准输入输出描述符 */
enum std_fd{
    stdin_no, //0 标准输入
    stdout_no, //1 标准输出
    stderr_no //2 标准错误
};
/* 位图类型 */
enum bitmap_type{
    INODE_BITMAP,  //inode位图
    BLOCK_BITMAP  //块位图
};
int get_free_slot_in_global(void);
int task_fd_install(int global_fd_idx);
int inode_bitmap_alloc(struct partition *part);
int block_bitmap_alloc(struct partition *part);
void bitmap_sync(struct partition *part,unsigned int bit_idx,unsigned char btmp);
int file_create(struct dir* parent_dir, char* filename, unsigned char flag);
int file_open(unsigned int inode_no, unsigned char flag);
int file_close(struct file* file);
int file_write(struct file* file, const void* buf, unsigned int count);
int file_read(struct file* file, void* buf, unsigned int count);
#endif
```



## fs.c

```
#include "ide-dev.h"
#include "fs.h"
#include "inode.h"
#include "super_block.h"
#include "dir.h"
#include "bitmap.h"
#include "file.h"
#include "../vga/vga.h"
#include "../mem/vmm.h"
#include "../asm/asm.h"
#include "../debug/debug.h"
#include "../task/task.h"
#include "../keyboard/keyboard.h"
#include "../sync/sync.h"
#include "../pipe/pipe.h"
#define NULL (void *)0
struct partition *cur_part;
extern struct task_struct *current;
extern struct dir *root_dir;             // 根目录
extern struct file file_table[MAX_FILE_OPEN];
extern char shell_input; //shell输入缓冲区
extern struct semaphore user_sema;
/* 
** 格式化分区，创建文件系统 
*/
static void partition_format(){
    //2048-9999扇区是主分区起始扇区和终止扇区
    struct partition *main_partition=read_main_partition();
    
    //超级块所占扇区数
    unsigned int super_block_sects=1;
    //inode位图所占扇区数
    unsigned int inode_bitmap_sects=(MAX_FILE+8*SEC_SIZE-1)/(8*SEC_SIZE);
    //inode结点表所占扇区数
    unsigned int inode_table_sects=((MAX_FILE*sizeof(struct inode))+8*SEC_SIZE-1)/(8*SEC_SIZE);
    unsigned int useful_sects=main_partition->sector_size-super_block_sects
                            -inode_bitmap_sects-inode_table_sects;
    /* 可用块位图所占扇区 */
    unsigned int block_bitmap_sects=(useful_sects+8*SEC_SIZE-1)/(8*SEC_SIZE);
    /* 空闲块数量 */
    unsigned int block_sects=useful_sects-block_bitmap_sects;
    
    /*   超级块初始化   */
    struct super_block sb;
    sb.magic=0x19590318;
    sb.sec_cnt=main_partition->sector_size;
    sb.inode_cnt=MAX_FILE;
    sb.part_lba_base=2048;  //超级块
    sb.block_bitmap_lba=2048+1;
    sb.block_bitmap_sects=block_bitmap_sects;

    sb.inode_bitmap_lba=sb.block_bitmap_lba+sb.block_bitmap_sects;
    sb.inode_bitmap_sects=inode_bitmap_sects;

    sb.inode_table_lba=sb.inode_bitmap_lba+sb.inode_bitmap_sects;
    sb.inode_table_sects=inode_table_sects;

    sb.data_start_lba=sb.inode_table_lba+sb.inode_table_sects;
    sb.root_inode_no=0;
    sb.dir_entry_size=sizeof(struct dir_entry);

    /* 打印数据 */
    clear();
    printk("sb.magic:%08ux\n",sb.magic);
    printk("sb.sec_cnt:%08ux\n",sb.sec_cnt);
    printk("sb.inode_cnt:%08ux\n",sb.inode_cnt);
    printk("sb.block_bitmap_lba:%08ux\n",sb.block_bitmap_lba);
    printk("sb.block_bitmap_sects:%08ux\n",sb.block_bitmap_sects);

    printk("sb.inode_bitmap_lba:%08ux\n",sb.inode_bitmap_lba);
    printk("sb.inode_bitmap_sects:%08ux\n",sb.inode_bitmap_sects);

    printk("sb.inode_table_lba:%08ux\n",sb.inode_table_lba);
    printk("sb.inode_table_sects:%08ux\n",sb.inode_table_sects);
    printk("sb.data_start_lba:%08ux\n",sb.data_start_lba); 
    /* 将超级块写入起始扇区-2048 */
    ide_write(&sb,2048,1); 
    /* 找到空闲块位图、inode位图和inode数组位图中的最大值,并算出其总字节数 */
    unsigned int buf_size=sb.block_bitmap_sects>sb.inode_bitmap_sects?
        sb.block_bitmap_sects:sb.inode_bitmap_sects;
    buf_size=(buf_size>sb.inode_table_sects?buf_size:sb.inode_table_sects)*SEC_SIZE;
    unsigned char *buf=(unsigned char *)vmm_malloc(buf_size,2);

    /* 初始化空闲块位图并写入空闲块位图起始扇区中 */
    buf[0] |= 0x01; //先给根目录占位

    /*将空闲块位图中多余的部分置位*/
    unsigned int last_byte=block_sects/8;
    unsigned char last_bit=block_sects%8;
    unsigned int last_size=SEC_SIZE-(last_byte%SEC_SIZE);

    /* 将超出实际块数的部分设置为占用 */
    memset(&buf[last_byte],0xFF,last_size);
    /* 把最后一字节内覆盖的空闲扇区置为0 */
    for(char i=0;i<last_bit;i++){
        buf[last_byte] &= ~(1<<i);
    }
    ide_write(buf,sb.block_bitmap_lba,sb.block_bitmap_sects);

    /* 初始化inode位图并写入inode位图起始扇区中 */
    memset(buf,0,buf_size);
    buf[0] |= 0x1; //第一个inode结点预留给根目录
    //处理最后一个扇区多余部分
    last_byte=MAX_FILE/8;
    last_bit=MAX_FILE%8;
    last_size=SEC_SIZE-last_byte;
    memset(&buf[last_byte],0xFF,last_size);
    for(char i=0;i<last_bit;i++){
        buf[last_byte] &= ~(1<<i);
    }
    ide_write(buf,sb.inode_bitmap_lba,sb.inode_bitmap_sects);

    /* 初始化inode数组并写入inode表位图起始扇区中 */
    memset(buf,0,buf_size);
    struct inode *i=(struct inode*)buf;
    i->i_size=sb.dir_entry_size*2; // . 和 ..
    i->i_num=0; //根目录占inode数组中第0个inode
    i->i_sectors[0]=sb.data_start_lba;
    ide_write(buf,sb.inode_table_lba,sb.inode_table_sects);

    /* 将根目录写入sb.data_start_lba */
    memset(buf,0,buf_size);

    struct dir_entry *p_de=(struct dir_entry*)buf;
    /* 初始化当前目录"." */
    memcpy(p_de->filename,".",1);
    p_de->i_no=0;
    p_de->f_type=FT_DIRECTORY;
    p_de++;

    /* 初始化当前目录父目录".." */
    memcpy(p_de->filename,"..",2);
    p_de->i_no=0;  //根目录的父目录依然是根目录
    p_de->f_type=FT_DIRECTORY;
    
    ide_write(buf,sb.data_start_lba,1);
    vmm_free(buf,buf_size);
}
/*
**   mount_partition():挂载分区，即将分区信息读入内存中
*/
void mount_partition(){
    cur_part=read_main_partition();
    struct super_block *sb_buf=(struct super_block *)vmm_malloc(SEC_SIZE,2);
    memset(sb_buf,0,sizeof(struct super_block));
    ide_read((unsigned char *)sb_buf,2048,1);

    /* 把sb_buf中超级块的信息复制到分区的超级块sb中*/
    cur_part->sb=sb_buf;

    /* 读取硬盘上的块位图信息 */
    cur_part->block_bitmap.bits=
    (unsigned char *)vmm_malloc(sb_buf->block_bitmap_sects*SEC_SIZE,2);
    cur_part->block_bitmap.btmp_bytes_len=sb_buf->block_bitmap_sects*SEC_SIZE;
    ide_read(cur_part->block_bitmap.bits,sb_buf->block_bitmap_lba,sb_buf->block_bitmap_sects);
    
    /* 读取硬盘上的inode位图信息 */
    cur_part->inode_bitmap.bits=
    (unsigned char *)vmm_malloc(sb_buf->inode_bitmap_sects*SEC_SIZE,2);
    cur_part->inode_bitmap.btmp_bytes_len=sb_buf->inode_bitmap_sects*SEC_SIZE;
    ide_read(cur_part->inode_bitmap.bits,sb_buf->inode_bitmap_lba,sb_buf->inode_bitmap_sects);

    /* 测试数据 */
    clear();
    
    printk("sb.magic:%08ux\n",cur_part->sb->magic);
    printk("sb.sec_cnt:%08ux\n",cur_part->sb->sec_cnt);
    printk("sb.inode_cnt:%08ux\n",cur_part->sb->inode_cnt);
    printk("sb.block_bitmap_lba:%08ux\n",cur_part->sb->block_bitmap_lba);
    printk("sb.block_bitmap_sects:%08ux\n",cur_part->sb->block_bitmap_sects);

    printk("sb.inode_bitmap_lba:%08ux\n",cur_part->sb->inode_bitmap_lba);
    printk("sb.inode_bitmap_sects:%08ux\n",cur_part->sb->inode_bitmap_sects);

    printk("sb.inode_table_lba:%08ux\n",cur_part->sb->inode_table_lba);
    printk("sb.inode_table_sects:%08ux\n",cur_part->sb->inode_table_sects);
    printk("sb.data_start_lba:%08ux\n",cur_part->sb->data_start_lba); 
    printk("cur_part->block_bitmap.bits:%08ux\n",cur_part->block_bitmap.bits);
    printk("cur_part->block_bitmap.btmp_bytes_len:%08ux\n",cur_part->block_bitmap.btmp_bytes_len);
    printk("cur_part->inode_bitmap.bits:%08ux\n",cur_part->inode_bitmap.bits);
    printk("cur_part->inode_bitmap.btmp_bytes_len:%08ux\n",cur_part->inode_bitmap.btmp_bytes_len);
    
    /* 将当前分区的根目录打开 */
   open_root_dir(cur_part);

   /* 初始化文件表 */
   unsigned int fd_idx = 0;
   while (fd_idx < MAX_FILE_OPEN) {
      file_table[fd_idx++].fd_inode = NULL;
   }
}
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

/* 返回路径深度,比如/a/b/c,深度为3 */
int path_depth_cnt(char* pathname) {
   ASSERT(pathname != NULL);
   char* p = pathname;
   char name[MAX_FILE_NAME_LEN];       // 用于path_parse的参数做路径解析
   unsigned int depth = 0;

   /* 解析路径,从中拆分出各级名称 */ 
   p = path_parse(p, name);
   while (name[0]) {
      depth++;
      memset(name, 0, MAX_FILE_NAME_LEN);
      if (p) {	     // 如果p不等于NULL,继续分析路径
	      p  = path_parse(p, name);
      }
   }
   return depth;
}

/* 搜索文件pathname,若找到则返回其inode号,否则返回-1 */
static int search_file(const char* pathname, struct path_search_record* searched_record) {
   /* 如果待查找的是根目录,为避免下面无用的查找,直接返回已知根目录信息 */
   if (!strcmp(pathname, "/") || !strcmp(pathname, "/.") || !strcmp(pathname, "/..")) {
      searched_record->parent_dir = root_dir;
      searched_record->file_type = FT_DIRECTORY;
      searched_record->searched_path[0] = 0;	   // 搜索路径置空
      return 0;
   }

   unsigned int path_len = strlen(pathname);
   /* 保证pathname至少是这样的路径/x且小于最大长度 */
   ASSERT(pathname[0] == '/' && path_len > 1 && path_len < MAX_PATH_LEN);
   char* sub_path = (char*)pathname;
   struct dir* parent_dir = root_dir;	
   struct dir_entry dir_e;

   /* 记录路径解析出来的各级名称,如路径"/a/b/c",
    * 数组name每次的值分别是"a","b","c" */
   char name[MAX_FILE_NAME_LEN] = {0};

   searched_record->parent_dir = parent_dir;
   searched_record->file_type = FT_UNKNOWN;
   unsigned int parent_inode_no = 0;  // 父目录的inode号
   
   sub_path = path_parse(sub_path, name);
   
   while (name[0]) {	   // 若第一个字符就是结束符,结束循环
      /* 记录查找过的路径,但不能超过searched_path的长度512字节 */
      ASSERT(strlen(searched_record->searched_path) < 512);

      /* 记录已存在的父目录 */
      strcat(searched_record->searched_path, "/");
      strcat(searched_record->searched_path, name);

      /* 在所给的目录中查找文件 */
      if (search_dir_entry(cur_part, parent_dir, name, &dir_e)) {         
         memset(name, 0, MAX_FILE_NAME_LEN);
         /* 若sub_path不等于NULL,也就是未结束时继续拆分路径 */
         if (sub_path) {
            sub_path = path_parse(sub_path, name);
         }

         if (FT_DIRECTORY == dir_e.f_type) {   // 如果被打开的是目录
            parent_inode_no = parent_dir->inode->i_num;
            dir_close(parent_dir);
            parent_dir = dir_open(cur_part, dir_e.i_no); // 更新父目录
            searched_record->parent_dir = parent_dir;
            continue;
         } else if (FT_REGULAR == dir_e.f_type) {	 // 若是普通文件
            searched_record->file_type = FT_REGULAR;
            return dir_e.i_no;
         }
      } else {		   //若找不到,则返回-1
         /* 找不到目录项时,要留着parent_dir不要关闭,
         * 若是创建新文件的话需要在parent_dir中创建 */
         return -1;
      }
   }

   /* 执行到此,必然是遍历了完整路径并且查找的文件或目录只有同名目录存在 */
   dir_close(searched_record->parent_dir);	      

   /* 保存被查找目录的直接父目录 */
   searched_record->parent_dir = dir_open(cur_part, parent_inode_no);	   
   searched_record->file_type = FT_DIRECTORY;
   return dir_e.i_no;
}
/* 打开或创建文件成功后,返回文件描述符,否则返回-1 */
int sys_open(const char* pathname, unsigned char flags) {
  /* 对目录要用dir_open,这里只有open文件 */
   if (pathname[strlen(pathname) - 1] == '/') {
      printk("can`t open a directory %s\n",pathname);
      return -1;
   }
   ASSERT(flags <= 7);
   int fd = -1;	   // 默认为找不到

   struct path_search_record searched_record;
   memset(&searched_record, 0, sizeof(struct path_search_record));

   /* 记录目录深度.帮助判断中间某个目录不存在的情况 */
   unsigned int pathname_depth = path_depth_cnt((char*)pathname);

   //printk("pathname : %s\n",pathname);
   /* 先检查文件是否存在 */
   int inode_no = search_file(pathname, &searched_record);
   char found = inode_no != -1 ? 1 : 0; 
  // printk("found : %02d\n",found);
   if (searched_record.file_type == FT_DIRECTORY) {
      printk("can`t open a direcotry with open(), use opendir() to instead\n");
      dir_close(searched_record.parent_dir);
      return -1;
   }

   unsigned int path_searched_depth = path_depth_cnt(searched_record.searched_path);

   /* 先判断是否把pathname的各层目录都访问到了,即是否在某个中间目录就失败了 */
   if (pathname_depth != path_searched_depth) {   // 说明并没有访问到全部的路径,某个中间目录是不存在的
      printk("cannot access %s: Not a directory, subpath %s is`t exist\n", \
	    pathname, searched_record.searched_path);
      dir_close(searched_record.parent_dir);
      return -1;
   }

   /* 若是在最后一个路径上没找到,并且并不是要创建文件,直接返回-1 */
   if (!found && !(flags & O_CREAT)) {
      printk("in path %s, file %s is`t exist\n", \
	    searched_record.searched_path, \
	    (strrchr(searched_record.searched_path, '/') + 1));
      dir_close(searched_record.parent_dir);
      return -1;
   } else if (found && flags & O_CREAT) {  // 若要创建的文件已存在
      printk("%s has already exist!\n", pathname);
      dir_close(searched_record.parent_dir);
      return -1;
   }

   switch (flags & O_CREAT) {
      case O_CREAT:
         printk("creating file\n");
         fd = file_create(searched_record.parent_dir, (strrchr(pathname, '/') + 1), flags);
         dir_close(searched_record.parent_dir);
         break;
      default:
         /* 其余情况均为打开已存在文件:
         * O_RDONLY,O_WRONLY,O_RDWR */
         //printk("inode_no:%08d\n",inode_no);
         fd = file_open(inode_no, flags);
   }

   /* 此fd是指任务task->fd_table数组中的元素下标,
    * 并不是指全局file_table中的下标 */
   return fd;
}
/* 将文件描述符转化为文件表的下标 */
unsigned int fd_local2global(unsigned int local_fd) {
   int global_fd = current->fd_table[local_fd];  
   ASSERT(global_fd >= 0 && global_fd < MAX_FILE_OPEN);
   return (unsigned int)global_fd;
} 

/* 关闭文件描述符fd指向的文件,成功返回0,否则返回-1 */
int sys_close(int fd) {
   int ret = -1;   // 返回值默认为-1,即失败
   if (fd > 2) {
      unsigned int global_fd = fd_local2global(fd);
      if (is_pipe(fd)) {
	 /* 如果此管道上的描述符都被关闭,释放管道的环形缓冲区 */
	 if (--file_table[global_fd].fd_pos == 0) {
	    vmm_free((unsigned int)file_table[global_fd].fd_inode, VMM_PAGE_SIZE);
	    file_table[global_fd].fd_inode = NULL;
	 }
	 ret = 0;
      } else {
	 ret = file_close(&file_table[global_fd]);
      }
      current->fd_table[fd] = -1; // 使该文件描述符位可用
   }
   return ret;
}
/* 将buf中连续count个字节写入文件描述符fd,成功则返回写入的字节数,失败返回-1 */
int sys_write(int fd, const void* buf, unsigned int count) {
   if (fd < 0) {
      printk("sys_write: fd error\n");
      return -1;
   }
   if (fd == stdout_no) { 
      /* 标准输出有可能被重定向为管道缓冲区, 因此要判断 */
      if (is_pipe(fd)) {
	 return pipe_write(fd, buf, count);
      } else { 
      char tmp_buf[1024] = {0};
      memcpy(tmp_buf, buf, count);
      printk(tmp_buf);
      printk("\n");
      return count;
      }
   }else if (is_pipe(fd)){	    /* 若是管道就调用管道的方法 */
      return pipe_write(fd, buf, count);
   } else {
   unsigned int _fd = fd_local2global(fd);
   struct file* wr_file = &file_table[_fd];
   if (wr_file->fd_flag & O_WRONLY || wr_file->fd_flag & O_RDWR) {
      unsigned int bytes_written  = file_write(wr_file, buf, count);
      return bytes_written;
   } else {
      printk("sys_write: not allowed to write file without flag O_RDWR or O_WRONLY\n");
      return -1;
   }
   }
}
/* 从文件描述符fd指向的文件中读取count个字节到buf,若成功则返回读出的字节数,到文件尾则返回-1 */
int sys_read(int fd, void* buf, unsigned int count) {
   ASSERT(buf != NULL);
   int ret = -1;
   if (fd < 0 || fd == stdout_no || fd == stderr_no) {
      printk("sys_read: fd error\n");
   } else if (fd == stdin_no) {
       /* 标准输入有可能被重定向为管道缓冲区, 因此要判断 */
      if (is_pipe(fd)) {
	 ret = pipe_read(fd, buf, count);
      } else {
      char* buffer = buf;
      unsigned int bytes_read = 0;
      if(shell_input==0){
         sema_down(&user_sema);
      }
         
      while (bytes_read < count) {
         *buffer = shell_input;//
         shell_input=0;
         bytes_read++;
         buffer++;
      }
      ret = (bytes_read == 0 ? -1 : (int)bytes_read);
      }
       } else if (is_pipe(fd)) {	 /* 若是管道就调用管道的方法 */
      ret = pipe_read(fd, buf, count);
   } else {
      unsigned int _fd = fd_local2global(fd);
      ret = file_read(&file_table[_fd], buf, count);   
   }
   //printk("ret=%08x\n",ret);
   return ret; 
}

/* 重置用于文件读写操作的偏移指针,成功时返回新的偏移量,出错时返回-1 */
int sys_lseek(int fd, int offset, unsigned char whence) {
   if (fd < 0) {
      printk("sys_lseek: fd error\n");
      return -1;
   }
   ASSERT(whence > 0 && whence < 4);
   unsigned int _fd = fd_local2global(fd);
   struct file* pf = &file_table[_fd];
   int new_pos = 0;   //新的偏移量必须位于文件大小之内
   int file_size = (int)pf->fd_inode->i_size;
   switch (whence) {
      /* SEEK_SET 新的读写位置是相对于文件开头再增加offset个位移量 */
      case SEEK_SET:
         new_pos = offset;
         break;
      /* SEEK_CUR 新的读写位置是相对于当前的位置增加offset个位移量 */
      case SEEK_CUR:	// offse可正可负
         new_pos = (int)pf->fd_pos + offset;
         break;
      /* SEEK_END 新的读写位置是相对于文件尺寸再增加offset个位移量 */
      case SEEK_END:	   // 此情况下,offset应该为负值
	      new_pos = file_size + offset;
   }
   if (new_pos < 0 || new_pos > (file_size - 1)) {	 
      return -1;
   }
   pf->fd_pos = new_pos;
   return pf->fd_pos;
}
/* 删除文件(非目录),成功返回0,失败返回-1 */
int sys_unlink(const char* pathname) {
   ASSERT(strlen(pathname) < MAX_PATH_LEN);

   /* 先检查待删除的文件是否存在 */
   struct path_search_record searched_record;
   memset(&searched_record, 0, sizeof(struct path_search_record));
   int inode_no = search_file(pathname, &searched_record);
   ASSERT(inode_no != 0);
   if (inode_no == -1) {
      printk("file %s not found!\n", pathname);
      dir_close(searched_record.parent_dir);
      return -1;
   }
   if (searched_record.file_type == FT_DIRECTORY) {
      printk("can`t delete a direcotry with unlink(), use rmdir() to instead\n");
      dir_close(searched_record.parent_dir);
      return -1;
   }

   /* 检查是否在已打开文件列表(文件表)中 */
   unsigned int file_idx = 0;
   while (file_idx < MAX_FILE_OPEN) {
      if (file_table[file_idx].fd_inode != NULL && (unsigned int)inode_no == file_table[file_idx].fd_inode->i_num) {
         break;
      }
      file_idx++;
   }
   if (file_idx < MAX_FILE_OPEN) {
      dir_close(searched_record.parent_dir);
      printk("file %s is in use, not allow to delete!\n", pathname);
      return -1;
   }
   ASSERT(file_idx == MAX_FILE_OPEN);
   
   /* 为delete_dir_entry申请缓冲区 */
   void* io_buf = vmm_malloc(SEC_SIZE*2,2);
   if (io_buf == NULL) {
      dir_close(searched_record.parent_dir);
      printk("sys_unlink: malloc for io_buf failed\n");
      return -1;
   }

   struct dir* parent_dir = searched_record.parent_dir;  
   delete_dir_entry(cur_part, parent_dir, inode_no, io_buf);
   inode_release(cur_part, inode_no);
   vmm_free(io_buf,SEC_SIZE*2);
   dir_close(searched_record.parent_dir);
   return 0;   // 成功删除文件 
}
/* 创建目录pathname,成功返回0,失败返回-1 */
int sys_mkdir(const char* pathname) {
   unsigned char rollback_step = 0;	       // 用于操作失败时回滚各资源状态
   unsigned int byte_idx,bit_odd;
   void* io_buf = vmm_malloc(SEC_SIZE * 2,2);
   if (io_buf == NULL) {
      printk("sys_mkdir: sys_malloc for io_buf failed\n");
      return -1;
   }

   struct path_search_record searched_record;
   memset(&searched_record, 0, sizeof(struct path_search_record));
   int inode_no = -1;
   inode_no = search_file(pathname, &searched_record);
   if (inode_no != -1) {      // 如果找到了同名目录或文件,失败返回
      printk("sys_mkdir: file or directory %s exist!\n", pathname);
      rollback_step = 1;
      goto rollback;
   } else {	     // 若未找到,也要判断是在最终目录没找到还是某个中间目录不存在
      unsigned int pathname_depth = path_depth_cnt((char*)pathname);
      unsigned int path_searched_depth = path_depth_cnt(searched_record.searched_path);
      /* 先判断是否把pathname的各层目录都访问到了,即是否在某个中间目录就失败了 */
      if (pathname_depth != path_searched_depth) {   // 说明并没有访问到全部的路径,某个中间目录是不存在的
         printk("sys_mkdir: can`t access %s, subpath %s is`t exist\n", pathname, searched_record.searched_path);
         rollback_step = 1;
         goto rollback;
      }
   }

   struct dir* parent_dir = searched_record.parent_dir;
   /* 目录名称后可能会有字符'/',所以最好直接用searched_record.searched_path,无'/' */
   char* dirname = strrchr(searched_record.searched_path, '/') + 1;

   inode_no = inode_bitmap_alloc(cur_part); 
   if (inode_no == -1) {
      printk("sys_mkdir: allocate inode failed\n");
      rollback_step = 1;
      goto rollback;
   }

   struct inode new_dir_inode;
   inode_init(inode_no, &new_dir_inode);	    // 初始化i结点

   unsigned int block_bitmap_idx = 0;     // 用来记录block对应于block_bitmap中的索引
   int block_lba = -1;
/* 为目录分配一个块,用来写入目录.和.. */
   block_lba = block_bitmap_alloc(cur_part);
   if (block_lba == -1) {
      printk("sys_mkdir: block_bitmap_alloc for create directory failed\n");
      rollback_step = 2;
      goto rollback;
   }
   new_dir_inode.i_sectors[0] = block_lba;
   /* 每分配一个块就将位图同步到硬盘 */
   block_bitmap_idx = block_lba - cur_part->sb->data_start_lba;
   ASSERT(block_bitmap_idx != 0);
   bitmap_sync(cur_part, block_bitmap_idx, BLOCK_BITMAP);
   
   /* 将当前目录的目录项'.'和'..'写入目录 */
   memset(io_buf, 0, SEC_SIZE * 2);	 // 清空io_buf
   struct dir_entry* p_de = (struct dir_entry*)io_buf;
   
   /* 初始化当前目录"." */
   memcpy(p_de->filename, ".", 1);
   p_de->i_no = inode_no ;
   p_de->f_type = FT_DIRECTORY;

   p_de++;
   /* 初始化当前目录".." */
   memcpy(p_de->filename, "..", 2);
   p_de->i_no = parent_dir->inode->i_num;
   p_de->f_type = FT_DIRECTORY;
   ide_write(io_buf, new_dir_inode.i_sectors[0], 1);

   new_dir_inode.i_size = 2 * cur_part->sb->dir_entry_size;

   /* 在父目录中添加自己的目录项 */
   struct dir_entry new_dir_entry;
   memset(&new_dir_entry, 0, sizeof(struct dir_entry));
   create_dir_entry(dirname, inode_no, FT_DIRECTORY, &new_dir_entry);
   memset(io_buf, 0, SEC_SIZE * 2);	 // 清空io_buf
   if (!sync_dir_entry(parent_dir, &new_dir_entry, io_buf)) {	  // sync_dir_entry中将block_bitmap通过bitmap_sync同步到硬盘
      printk("sys_mkdir: sync_dir_entry to disk failed!\n");
      rollback_step = 2;
      goto rollback;
   }

   /* 父目录的inode同步到硬盘 */
   memset(io_buf, 0, SEC_SIZE * 2);
   inode_sync(cur_part, parent_dir->inode, io_buf);

   /* 将新创建目录的inode同步到硬盘 */
   memset(io_buf, 0, SEC_SIZE * 2);
   inode_sync(cur_part, &new_dir_inode, io_buf);

   /* 将inode位图同步到硬盘 */
   bitmap_sync(cur_part, inode_no, INODE_BITMAP);

   vmm_free(io_buf,SEC_SIZE * 2);

   /* 关闭所创建目录的父目录 */
   dir_close(searched_record.parent_dir);
   return 0;

/*创建文件或目录需要创建相关的多个资源,若某步失败则会执行到下面的回滚步骤 */
rollback:	     // 因为某步骤操作失败而回滚
   switch (rollback_step) {
      case 2:
         byte_idx = inode_no / 8;
         bit_odd  = inode_no % 8;
         cur_part->inode_bitmap.bits[byte_idx] &= ~(1 << bit_odd);
	 case 1:
         /* 关闭所创建目录的父目录 */
         dir_close(searched_record.parent_dir);
         break;
   }
   vmm_free(io_buf,SEC_SIZE*2);
   return -1;
}
/* 目录打开成功后返回目录指针,失败返回NULL */
struct dir* sys_opendir(const char* name) {
   ASSERT(strlen(name) < MAX_PATH_LEN);
   /* 如果是根目录'/',直接返回&root_dir */
   if (name[0] == '/' && (name[1] == 0 || name[0] == '.')) {
      return root_dir;
   }

   /* 先检查待打开的目录是否存在 */
   struct path_search_record searched_record;
   memset(&searched_record, 0, sizeof(struct path_search_record));
   int inode_no = search_file(name, &searched_record);
   struct dir* ret = NULL;
   if (inode_no == -1) {	 // 如果找不到目录,提示不存在的路径 
      printk("In %s, sub path %s not exist\n", name, searched_record.searched_path); 
   } else {
      if (searched_record.file_type == FT_REGULAR) {
	      printk("%s is regular file!\n", name);
      } else if (searched_record.file_type == FT_DIRECTORY) {
	      ret = dir_open(cur_part, inode_no);
      }
   }
   dir_close(searched_record.parent_dir);
   return ret;
}

/* 成功关闭目录dir返回0,失败返回-1 */
int sys_closedir(struct dir* dir) {
   int ret = -1;
   if (dir != NULL) {
      dir_close(dir);
      ret = 0;
   }
   return ret;
}
/* 读取目录dir的1个目录项,成功后返回其目录项地址,到目录尾时或出错时返回NULL */
struct dir_entry* sys_readdir(struct dir* dir) {
   ASSERT(dir != NULL);
   return dir_read(dir);
}

/* 把目录dir的指针dir_pos置0 */
void sys_rewinddir(struct dir* dir) {
   dir->dir_pos = 0;
}
/* 删除空目录,成功时返回0,失败时返回-1*/
int sys_rmdir(const char* pathname) {
   /* 先检查待删除的文件是否存在 */
   struct path_search_record searched_record;
   memset(&searched_record, 0, sizeof(struct path_search_record));
   int inode_no = search_file(pathname, &searched_record);
   ASSERT(inode_no != 0);
   int retval = -1;	// 默认返回值
   if (inode_no == -1) {
      printk("In %s, sub path %s not exist\n", pathname, searched_record.searched_path); 
   } else {
      if (searched_record.file_type == FT_REGULAR) {
         printk("%s is regular file!\n", pathname);
      } else { 
         struct dir* dir = dir_open(cur_part, inode_no);
         if (!dir_is_empty(dir)) {	 // 非空目录不可删除
            printk("dir %s is not empty, it is not allowed to delete a nonempty directory!\n", pathname);
         } else {
            if (!dir_remove(searched_record.parent_dir, dir)) {
               retval = 0;
            }
         }
         dir_close(dir);
      }
   }
   dir_close(searched_record.parent_dir);
   return retval;
}
/* 获得父目录的inode编号 */
static unsigned int get_parent_dir_inode_nr(unsigned int child_inode_nr, void* io_buf) {
   struct inode* child_dir_inode = inode_open(cur_part, child_inode_nr);
   /* 目录中的目录项".."中包括父目录inode编号,".."位于目录的第0块 */
   unsigned int block_lba = child_dir_inode->i_sectors[0];
   ASSERT(block_lba >= cur_part->sb->data_start_lba);
   inode_close(child_dir_inode);
   ide_read(io_buf, block_lba,  1);   
   struct dir_entry* dir_e = (struct dir_entry*)io_buf;
   /* 第0个目录项是".",第1个目录项是".." */
   ASSERT(dir_e[1].i_no < 4096 && dir_e[1].f_type == FT_DIRECTORY);
   return dir_e[1].i_no;      // 返回..即父目录的inode编号
}

/* 在inode编号为p_inode_nr的目录中查找inode编号为c_inode_nr的子目录的名字,
 * 将名字存入缓冲区path.成功返回0,失败返-1 */
static int get_child_dir_name(unsigned int p_inode_nr, unsigned int c_inode_nr, char* path, void* io_buf) {
   struct inode* parent_dir_inode = inode_open(cur_part, p_inode_nr);
   /* 填充all_blocks,将该目录的所占扇区地址全部写入all_blocks */
   unsigned char block_idx = 0;
   unsigned int all_blocks[140] = {0}, block_cnt = 12;
   while (block_idx < 12) {
      all_blocks[block_idx] = parent_dir_inode->i_sectors[block_idx];
      block_idx++;
   }
   if (parent_dir_inode->i_sectors[12]) {	// 若包含了一级间接块表,将共读入all_blocks.
      ide_read(all_blocks + 12,parent_dir_inode->i_sectors[12],  1);
      block_cnt = 140;
   }
   inode_close(parent_dir_inode);

   struct dir_entry* dir_e = (struct dir_entry*)io_buf;
   unsigned int dir_entry_size = cur_part->sb->dir_entry_size;
   unsigned int dir_entrys_per_sec = (512 / dir_entry_size);
   block_idx = 0;
  /* 遍历所有块 */
   while(block_idx < block_cnt) {
      if(all_blocks[block_idx]) {      // 如果相应块不为空则读入相应块
         ide_read(io_buf, all_blocks[block_idx],  1);
         unsigned char dir_e_idx = 0;
         /* 遍历每个目录项 */
         while(dir_e_idx < dir_entrys_per_sec) {
            if ((dir_e + dir_e_idx)->i_no == c_inode_nr) {
               strcat(path, "/");
               strcat(path, (dir_e + dir_e_idx)->filename);
               return 0;
            }
            dir_e_idx++;
         }
      }
      block_idx++;
   }
   return -1;
}

/* 把当前工作目录绝对路径写入buf, size是buf的大小. 
 当buf为NULL时,由操作系统分配存储工作路径的空间并返回地址
 失败则返回NULL */
char* sys_getcwd(char* buf, unsigned int size) {
   /* 确保buf不为空,若用户进程提供的buf为NULL,
   系统调用getcwd中要为用户进程通过malloc分配内存 */
   ASSERT(buf != NULL);
   void* io_buf = vmm_malloc(SEC_SIZE,2);
   if (io_buf == NULL) {
      return NULL;
   }

   int parent_inode_nr = 0;
   int child_inode_nr = current->cwd_inode_nr;
   ASSERT(child_inode_nr >= 0 && child_inode_nr < 4096);      // 最大支持4096个inode
   /* 若当前目录是根目录,直接返回'/' */
   if (child_inode_nr == 0) {
      buf[0] = '/';
      buf[1] = 0;
      return buf;
   }  

   memset(buf, 0, size);
   char full_path_reverse[MAX_PATH_LEN] = {0};	  // 用来做全路径缓冲区

   /* 从下往上逐层找父目录,直到找到根目录为止.
    * 当child_inode_nr为根目录的inode编号(0)时停止,
    * 即已经查看完根目录中的目录项 */
   while ((child_inode_nr)) {
      parent_inode_nr = get_parent_dir_inode_nr(child_inode_nr, io_buf);
      if (get_child_dir_name(parent_inode_nr, child_inode_nr, full_path_reverse, io_buf) == -1) {	  // 或未找到名字,失败退出
         vmm_free(io_buf,SEC_SIZE);
         return NULL;
      }
      child_inode_nr = parent_inode_nr;
   }
   ASSERT(strlen(full_path_reverse) <= size);
/* 至此full_path_reverse中的路径是反着的,
 * 即子目录在前(左),父目录在后(右) ,
 * 现将full_path_reverse中的路径反置 */
   char* last_slash;	// 用于记录字符串中最后一个斜杠地址
   while ((last_slash = strrchr(full_path_reverse, '/'))) {
      unsigned short len = strlen(buf);
      strcpy(buf + len, last_slash);
      /* 在full_path_reverse中添加结束字符,做为下一次执行strcpy中last_slash的边界 */
      *last_slash = 0;
   }
   vmm_free(io_buf,SEC_SIZE);
   return buf;
}
/* 在buf中填充文件结构相关信息,成功时返回0,失败返回-1 */
int sys_stat(const char* path, struct stat* buf) {
   /* 若直接查看根目录'/' */
   if (!strcmp(path, "/") || !strcmp(path, "/.") || !strcmp(path, "/..")) {
      buf->st_filetype = FT_DIRECTORY;
      buf->st_ino = 0;
      buf->st_size = root_dir->inode->i_size;
      return 0;
   }

   int ret = -1;	// 默认返回值
   struct path_search_record searched_record;
   memset(&searched_record, 0, sizeof(struct path_search_record));   // 记得初始化或清0,否则栈中信息不知道是什么
   int inode_no = search_file(path, &searched_record);
   if (inode_no != -1) {
      struct inode* obj_inode = inode_open(cur_part, inode_no);   // 只为获得文件大小
      buf->st_size = obj_inode->i_size;
      inode_close(obj_inode);
      buf->st_filetype = searched_record.file_type;
      buf->st_ino = inode_no;
      ret = 0;
   } else {
      printk("sys_stat: %s not found\n", path);
   }
   dir_close(searched_record.parent_dir);
   return ret;
}

/* 更改当前工作目录为绝对路径path,成功则返回0,失败返回-1 */
int sys_chdir(const char* path) {
   int ret = -1;
   struct path_search_record searched_record;  
   memset(&searched_record, 0, sizeof(struct path_search_record));
   int inode_no = search_file(path, &searched_record);
   if (inode_no != -1) {
      if (searched_record.file_type == FT_DIRECTORY) {
         current->cwd_inode_nr = inode_no;
         ret = 0;
      } else {
         printk("sys_chdir: %s is regular file or other!\n", path);
      }
   }
   dir_close(searched_record.parent_dir); 
   return ret;
}

void fs_init(){
    //partition_format();  //初始化主分区，只需运行一次就行，硬盘中就有分区信息
    mount_partition();
    //unsigned int fd=sys_open("/file",O_CREAT);
   // unsigned int fd=sys_open("/file",O_RDWR); //O_RDONLY
    //sys_open("/file1",O_CREAT);
    //sys_open("/file2",O_CREAT);
    //sys_open("/file",O_CREAT);
    //printk("fd:%02d\n",fd);
    //printk("open /file1 fd1:%02d\n",fd1);
    //sys_write(fd,"Hello,world!Hello,world!\n",25);
   // char buf[64]={0};
    //sys_lseek(fd,0,SEEK_SET);
    //int read_bytes=sys_read(fd,buf,13);
    //printk("read %08x bytes:\n %s\n",read_bytes,buf);
    //memset(buf,0,64);
    //read_bytes=sys_read(fd,buf,12);
    //printk("read %08x bytes:\n %s\n",read_bytes,buf);
    //sys_lseek(fd,0,SEEK_SET);
    //memset(buf,0,64);
    //read_bytes=sys_read(fd,buf,12);
    //printk("read %08x bytes:\n %s\n",read_bytes,buf);
    //sys_close(fd);
    //printk("%02d closed now\n",fd1);
    //printk("/file1 delete %s!\n", sys_unlink("/file") == 0 ? "done" : "fail");
   
    //测试目录
  /*  printk("/dir1/subdir1 create %s!\n", sys_mkdir("/dir1/subdir1") == 0 ? "done" : "fail");
   printk("/dir1 create %s!\n", sys_mkdir("/dir1") == 0 ? "done" : "fail");
   printk("now, /dir1/subdir1 create %s!\n", sys_mkdir("/dir1/subdir1") == 0 ? "done" : "fail");
   int fd = sys_open("/dir1/subdir1/file2", O_CREAT|O_RDWR);
   if (fd != -1) {
      printk("/dir1/subdir1/file2 create done!\n");
      sys_write(fd, "Catch me if you can!\n", 21);
      sys_lseek(fd, 0, SEEK_SET);
      char buf[32] = {0};
      sys_read(fd, buf, 21); 
      printk("/dir1/subdir1/file2 says:\n%s", buf);
      sys_close(fd);
   } */
   //测试打开目录
   /*   struct dir* p_dir = sys_opendir("/dir1/subdir1");
   if (p_dir) {
      printk("/dir1/subdir1 open done!\n");
      if (sys_closedir(p_dir) == 0) {
	 printk("/dir1/subdir1 close done!\n");
      } else {
	 printk("/dir1/subdir1 close fail!\n");
      }
   } else {
      printk("/dir1/subdir1 open fail!\n");
   }
   */
  //测试读目录
  /*struct dir* p_dir = sys_opendir("/dir1/subdir1");
   if (p_dir) {
      printk("/dir1/subdir1 open done!\ncontent:\n");
      char* type = NULL;
      struct dir_entry* dir_e = NULL;
      while((dir_e = sys_readdir(p_dir))) { 
	 if (dir_e->f_type == FT_REGULAR) {
	    type = "regular";
	 } else {
	    type = "directory";
	 }
	 printk("      %s   %s\n", type, dir_e->filename);
      }
      if (sys_closedir(p_dir) == 0) {
	 printk("/dir1/subdir1 close done!\n");
      } else {
	 printk("/dir1/subdir1 close fail!\n");
      }
   } else {
      printk("/dir1/subdir1 open fail!\n");
   }
   */
  //测试删除目录
  /*clear();
   printk("/dir1 content before delete /dir1/subdir1:\n");
   struct dir* dir = sys_opendir("/dir1/");
   char* type = NULL;
   struct dir_entry* dir_e = NULL;
   while((dir_e = sys_readdir(dir))) { 
      if (dir_e->f_type == FT_REGULAR) {
	 type = "regular";
      } else {
	 type = "directory";
      }
      printk("      %s   %s\n", type, dir_e->filename);
   }
   printk("try to delete nonempty directory /dir1/subdir1\n");
   if (sys_rmdir("/dir1/subdir1") == -1) {
      printk("sys_rmdir: /dir1/subdir1 delete fail!\n");
   }

   printk("try to delete /dir1/subdir1/file2\n");
   if (sys_rmdir("/dir1/subdir1/file2") == -1) {
      printk("sys_rmdir: /dir1/subdir1/file2 delete fail!\n");
   } 
   if (sys_unlink("/dir1/subdir1/file2") == 0 ) {
      printk("sys_unlink: /dir1/subdir1/file2 delete done\n");
   }
   
   printk("try to delete directory /dir1/subdir1 again\n");
   if (sys_rmdir("/dir1/subdir1") == 0) {
      printk("/dir1/subdir1 delete done!\n");
   }

   printk("/dir1 content after delete /dir1/subdir1:\n");
   sys_rewinddir(dir);
   while((dir_e = sys_readdir(dir))) { 
      if (dir_e->f_type == FT_REGULAR) {
	 type = "regular";
      } else {
	 type = "directory";
      }
      printk("      %s   %s\n", type, dir_e->filename);
   }
   */
  //测试打开工作目录
   /*char cwd_buf[32] = {0};
   sys_getcwd(cwd_buf, 32);
   printk("cwd:%s\n", cwd_buf);
   sys_chdir("/dir1");
   printk("change cwd now\n");
   sys_getcwd(cwd_buf, 32);
   printk("cwd:%s\n", cwd_buf);*/
   //获得文件属性
  /* struct stat obj_stat;
   clear();
   sys_stat("/", &obj_stat);
   printk("/`s info\n   i_no:%02x\n   size:%02x\n   filetype:%s\n", \
	 obj_stat.st_ino, obj_stat.st_size, \
	 obj_stat.st_filetype == 2 ? "directory" : "regular");
   sys_stat("/dir1", &obj_stat);
   printk("/dir1`s info\n   i_no:%02x\n   size:%02x\n   filetype:%s\n", \
	 obj_stat.st_ino, obj_stat.st_size, \
	 obj_stat.st_filetype == 2 ? "directory" : "regular");*/
}
```



## fs.h

```
#ifndef _FS_H_
#define _FS_H_
#define MAX_FILE 1000  //一个分区最大文件数
#define SEC_SIZE 512  //扇区大小
#define MAX_PATH_LEN 512	    // 路径最大长度
/* 文件类型 */
enum file_types {
   FT_UNKNOWN,	  // 不支持的文件类型
   FT_REGULAR,	  // 普通文件
   FT_DIRECTORY	  // 目录
};

/* 打开文件的选项 */
enum oflags {
   O_RDONLY,	  // 只读
   O_WRONLY,	  // 只写
   O_RDWR,	  // 读写
   O_CREAT = 4	  // 创建
};

/* 文件读写位置偏移量 */
enum whence {
   SEEK_SET = 1,
   SEEK_CUR,
   SEEK_END
};

/* 文件属性结构体 */
struct stat {
   unsigned int st_ino;		 // inode编号
   unsigned int st_size;		 // 尺寸
   enum file_types st_filetype;	 // 文件类型
};
/* 用来记录查找文件过程中已找到的上级路径,也就是查找文件过程中"走过的地方" */
struct path_search_record {
   char searched_path[MAX_PATH_LEN];	    // 查找过程中的父路径
   struct dir* parent_dir;		    // 文件或目录所在的直接父目录
   enum file_types file_type;		    // 找到的是普通文件还是目录,找不到将为未知类型(FT_UNKNOWN)
};
void mount_partition();
int path_depth_cnt(char* pathname);
unsigned int fd_local2global(unsigned int local_fd);
int sys_open(const char* pathname, unsigned char flags);
int sys_close(int fd);
int sys_write(int fd, const void* buf, unsigned int count);
int sys_read(int fd, void* buf, unsigned int count);
int sys_lseek(int fd, int offset, unsigned char whence);
int sys_unlink(const char* pathname);
int sys_mkdir(const char* pathname);
struct dir_entry* sys_readdir(struct dir* dir);
void sys_rewinddir(struct dir* dir);
int sys_rmdir(const char* pathname);
char* sys_getcwd(char* buf, unsigned int size);
int sys_stat(const char* path, struct stat* buf);
int sys_chdir(const char* path);
struct dir* sys_opendir(const char* name);
int sys_closedir(struct dir* dir);
struct dir_entry* sys_readdir(struct dir* dir);
void fs_init();
#endif
```



## ide-dev.c

```
#include "ide-dev.h"
#include "../asm/asm.h"
#include "../vga/vga.h"
#include "../mem/vmm.h"
extern unsigned int init_end;
extern unsigned int kernel_bss;

/*
*   waitdisk:等待硬盘准备好
*/
static inline void waitdisk(void) {
    while ((inb(0x1F7) & 0xC0) != 0x40)
        ;
}

/*
*insl(port,addr,cnt):从port端口循环读cnt次双字到addr位置
*
*cld指令是使DF=0， 即si，di寄存器自动增加
*
*rep指令的目的是重复其上面的指令.ECX的值是重复的次数.repe和repne，
*前者是repeat equal，意思是相等的时候重复，后者是repeat not equal，
*不等的时候重复；每循环一次cx自动减一。
*
*insl 指令是从 DX 指定的 I/O 端口将双字输入 ES:(E)DI 指定的内存位置
*
*/
static inline void insl(unsigned int port, void *addr, int cnt) {
    asm volatile (
            "cld;"
            "repne; insl;"
            : "=D" (addr), "=c" (cnt)
            : "d" (port), "0" (addr), "1" (cnt)
            : "memory", "cc");
}

/*
**  outsl(port,addr,cnt):从port端口循环写cnt次双字到addr位置
*/
static inline void outsl(unsigned int port, void *addr, int cnt) {
    asm volatile (
            "cld;"
            "repne; outsl;"
            : "=S" (addr), "=c" (cnt)
            : "d" (port), "0" (addr), "1" (cnt)
            : "memory", "cc");
}

/*
**   ide_read_sect(dst,secno):读取扇区号secno所在的扇区进入dst地址中
*/
static void ide_read_sect(void *dst, unsigned int secno) {
    
    // 等待硬盘准备好
    waitdisk();

    outb(0x1F2, 1);                  // count = 1  读一个扇区
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
    outb(0x1F7, 0x20);               // 命令0x20 - 读取扇区 

    // 等待硬盘准备好
    waitdisk();
    // 读取一个扇区
    insl(0x1F0, dst, SECTSIZE / 4);
}

/*
**   ide_write_sect(src,secno):将src的数据写入到secno扇区中
*/
static void ide_write_sect(void *src, unsigned int secno) {
    
    // 等待硬盘准备好
    waitdisk();

    outb(0x1F2, 1);                  // count = 1 读一个扇区
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
    outb(0x1F7, 0x30);               // 命令0x30 - 存取扇区 

    // 等待硬盘准备好
    waitdisk();
    // 读取一个扇区
    outsl(0x1F0, src, SECTSIZE / 4);
} 

/*
**    ide_read(dst,secno,size):将secno开始的um个扇区读入dst起始的size大小的字节中
*/
void ide_read(void *dst,unsigned int secno,unsigned int num){
   // unsigned int num=(size+SECTSIZE-1)/SECTSIZE;
    for(int i=0;i<num;i++){
        ide_read_sect(dst,secno+i);
        dst+=SECTSIZE;
    }
}

/*
**    ide_write(src,secno,size):将src起始的num个扇区写入secno开始的扇区中
*/
void ide_write(void *src,unsigned int secno,unsigned int num){
    //unsigned int num=(size+SECTSIZE-1)/SECTSIZE;
    for(int i=0;i<num;i++){
        ide_write_sect(src,secno+i);
        src+=SECTSIZE;
    }
}
// 定义一个延时xus毫秒的延时函数
static void delay(unsigned int xus) // xus代表需要延时的微秒数
{
    unsigned int x,y;
    for(x=xus;x>0;x--)
        for(y=110;y>0;y--);
}
//硬盘默认只有一个主分区
struct partition *read_main_partition(){
    struct partition *main_partition;
    //申请内存存储主分区信息
    unsigned int main_part=vmm_malloc(SECTSIZE,2);
    //主分区位于0号扇区
    ide_read_sect((unsigned char *)main_part,0);
    //分区表位于引导扇区（0号扇区）的0x1BE - 0x1FD中
    //由于只有一个主分区，所以读取0x1BE开始的16个字节数据
    main_partition=(struct partition *)((unsigned char *)main_part+0x1BE);

    return main_partition;

}
//测试硬盘读写
void test_ide_io(){
    printk("INIT END:%08X ",&init_end);
    printk("BSS START:%08X",&kernel_bss);
    
    //init部分所占扇区
    int init_sec=(((unsigned int)&init_end-(unsigned int)0x100000)+512-1)/(512);
    //kernel部分所占扇区 ,其中bss段不占硬盘
    int kernel_sec=(((unsigned int)&kernel_bss-(unsigned int)0xC1000000)+512-1)/(512);
    //可用起始扇区，可能会覆盖.debug节、.stab、.stabstr、.symtab节等
    int start_sec=init_sec+kernel_sec;
    printk("start_sec:%08d ",start_sec);
    start_sec++;   //MBR也会占用一个扇区
    //已经将KERNEL读入内核，无需硬盘
    start_sec=0;
   /* unsigned int start=vmm_malloc(512,1);
    for(int i=0;i<512;i++){
        *((unsigned int *)start+i)=0xF10F15F2;
    }
    printk("start data:%08ux",*((unsigned int *)start+1));
    ide_write_sect((unsigned int *)start,start_sec);
  //  delay(5000000);
  
    unsigned int read=vmm_malloc(512,1);
    ide_read_sect((unsigned int *)read,start_sec);
    printk("read data:%08ux",*(unsigned int *)read);*/
    struct partition *main_partition=read_main_partition();
}

```



## ide-dev.h

```
#ifndef _IDE_DEV_H_
#define _IDE_DEV_H_
#include "bitmap.h"
#include "super_block.h"
#define SECTSIZE        512

/* 分区表项结构 */
struct partition
{
    unsigned char active_flag; //活动分区标记，0x80代表活动分区，0代表非活动分区
    unsigned char start_magnetic; //分区起始磁头号
    unsigned char start_sector; //分区起始扇区号
    unsigned char start_cylinder; //分区起始柱面号
    unsigned char file_type; //文件系统类型ID，0表示不可识别，0x83表示linux文件系统
    unsigned char end_magnetic; //分区结束磁头号
    unsigned char end_sector; //分区结束扇区号
    unsigned char end_cylinder; //分区结束柱面号
    unsigned int  start_offset; //分区起始偏移扇区
    unsigned int  sector_size; //分区扇区总数
    struct super_block *sb;  //该分区超级块
    struct bitmap block_bitmap; //空闲块位图
    struct bitmap inode_bitmap; //inode节点位图
}__attribute__((packed));

void ide_read(void *dst,unsigned int secno,unsigned int num);
void ide_write(void *src,unsigned int secno,unsigned int num);
void test_ide_io();
struct partition *read_main_partition();

#endif

```



## inode.c

```
#include "inode.h"
#include "ide-dev.h"
#include "file.h"
#include "../asm/asm.h"
#include "../mem/vmm.h"
#include "../debug/debug.h"
#define NULL (void *)0
extern struct partition *cur_part;
/* 用来存储inode位置 */
struct inode_position{
    char two_sec; //inode是否跨扇区
    unsigned int sec_lba; //inode所在扇区号
    unsigned int off_size; //inode在扇区内的字节偏移量
};
/* 获取inode所在的扇区和扇区内的偏移量 */
static void inode_locate(struct partition *part,unsigned int inode_no,struct inode_position *inode_pos){
    
    /* inode_table在硬盘上是连续的 */
    unsigned int inode_table_lba=part->sb->inode_table_lba;
    
    unsigned int inode_size=sizeof(struct inode);
    //第inode_no号inode节点相对于inode_table_lba的字节偏移量
    unsigned int off_size=inode_no * inode_size;

    //第inode_no号inode节点相对于inode_table_lba的扇区偏移量
    unsigned int off_sec=off_size / 512;
    //待查找的inode所在扇区中的起始地址
    unsigned int off_size_in_sec=off_size % 512;

    /* 判断i节点是否跨越了2个扇区 */
    unsigned int left_in_sec=SECTSIZE-off_size_in_sec;
    //若扇区内剩下的空间不足以容纳一个inode，必然是inode节点跨越了2个扇区
    if(left_in_sec < inode_size){
        inode_pos->two_sec=1; 
    }
    else{
        inode_pos->two_sec=0;
    }
    inode_pos->sec_lba=inode_table_lba+off_sec;
    inode_pos->off_size=off_size_in_sec;

}
/* 将inode写入到分区 */
void inode_sync(struct partition *part,struct inode *inode,void *io_buf){
    unsigned char inode_no=inode->i_num;
    struct inode_position inode_pos;
    //将inode位置信息存入inode_pos
    inode_locate(part,inode_no,&inode_pos);

    struct inode pure_inode;
    memcpy(&pure_inode,inode,sizeof(struct inode));

    /* 以下inode的两个成员只存在于内存中 */
    pure_inode.i_open_cnts=0;
    pure_inode.write_lock=0;
    //pure_inode.inode_tag.prev=pure_inode.inode_tag.next=NULL;
    char *inode_buf=(char *)io_buf;
    //若是跨了两个扇区，就要读出两个扇区再写入两个扇区
    if(inode_pos.two_sec){
        ide_read(inode_buf,inode_pos.sec_lba,2);
        memcpy(inode_buf+inode_pos.off_size,&pure_inode,sizeof(struct inode));
        ide_write(inode_buf,inode_pos.sec_lba,2);
    }
    else{  //只有一个扇区
        ide_read(inode_buf,inode_pos.sec_lba,1);
        memcpy(inode_buf+inode_pos.off_size,&pure_inode,sizeof(struct inode));
        ide_write(inode_buf,inode_pos.sec_lba,1);
    }
    
}
/* 根据i节点号返回相应的i节点 */
struct inode* inode_open(struct partition *part,unsigned int inode_no){

    /* 从硬盘中读入此inode */
    struct inode_position inode_pos;
    /* 将inode位置信息存入inode_pos */
    inode_locate(part,inode_no,&inode_pos);
    /* 创建新节点 */
    struct inode *inode_found=(struct inode *)vmm_malloc(sizeof(struct inode),2);
    char *inode_buf;
    //如果节点号在硬盘中是跨扇区的
    if(inode_pos.two_sec){
        inode_buf = (char *)vmm_malloc(SECTSIZE*2,2);
        ide_read(inode_buf,inode_pos.sec_lba,2);
    }
    else{
        inode_buf = (char *)vmm_malloc(SECTSIZE,2);
        ide_read(inode_buf,inode_pos.sec_lba,1);       
    }
    memcpy(inode_found,inode_buf+inode_pos.off_size,sizeof(struct inode));
    if(inode_pos.two_sec){
        vmm_free(inode_buf,SECTSIZE*2);
    }
    else{
        vmm_free(inode_buf,SECTSIZE);
    }
    return inode_found;
}
/* 关闭inode或减少inode的打开数 */
void inode_close(struct inode *inode){
    /* 若没有进程再打开此文件，将此inode释放空间 */
    vmm_free(inode,sizeof(struct inode));
}
/* 将硬盘分区part上的inode清空 */
void inode_delete(struct partition* part, unsigned int inode_no, void* io_buf) {
   ASSERT(inode_no < 4096);
   struct inode_position inode_pos;
   inode_locate(part, inode_no, &inode_pos);     // inode位置信息会存入inode_pos
   ASSERT(inode_pos.sec_lba <= (part->start_sector + part->sector_size));
   
   char* inode_buf = (char*)io_buf;
   if (inode_pos.two_sec) {   // inode跨扇区,读入2个扇区
      /* 将原硬盘上的内容先读出来 */
      ide_read(inode_buf, inode_pos.sec_lba,  2);
      /* 将inode_buf清0 */
      memset((inode_buf + inode_pos.off_size), 0, sizeof(struct inode));
      /* 用清0的内存数据覆盖磁盘 */
      ide_write(inode_buf, inode_pos.sec_lba,  2);
   } else {    // 未跨扇区,只读入1个扇区就好
      /* 将原硬盘上的内容先读出来 */
      ide_read(inode_buf, inode_pos.sec_lba, 1);
      /* 将inode_buf清0 */
      memset((inode_buf + inode_pos.off_size), 0, sizeof(struct inode));
      /* 用清0的内存数据覆盖磁盘 */
      ide_write(inode_buf, inode_pos.sec_lba,  1);
   }
}

/* 回收inode的数据块和inode本身 */
void inode_release(struct partition* part, unsigned int inode_no) {
   struct inode* inode_to_del = inode_open(part, inode_no);
   ASSERT(inode_to_del->i_num == inode_no);

    // 向下取整用于索引数组下标
    unsigned int byte_idx;
    // 取余用于索引数组内的位
    unsigned int bit_odd;
/* 1 回收inode占用的所有块 */
   unsigned char block_idx = 0, block_cnt = 12;
   unsigned int block_bitmap_idx;
   unsigned int all_blocks[140] = {0};	  //12个直接块+128个间接块

   /* a 先将前12个直接块存入all_blocks */
   while (block_idx < 12) {
      all_blocks[block_idx] = inode_to_del->i_sectors[block_idx];
      block_idx++;
   }

   /* b 如果一级间接块表存在,将其128个间接块读到all_blocks[12~], 并释放一级间接块表所占的扇区 */
   if (inode_to_del->i_sectors[12] != 0) {
      ide_read(all_blocks + 12, inode_to_del->i_sectors[12],  1);
      block_cnt = 140;

      /* 回收一级间接块表占用的扇区 */
      block_bitmap_idx = inode_to_del->i_sectors[12] - part->sb->data_start_lba;
      ASSERT(block_bitmap_idx > 0);
      byte_idx = block_bitmap_idx / 8;
      bit_odd  = block_bitmap_idx % 8;
      part->block_bitmap.bits[byte_idx] &= ~(1 << bit_odd);
      bitmap_sync(cur_part, block_bitmap_idx, BLOCK_BITMAP);
   }
   
   /* c inode所有的块地址已经收集到all_blocks中,下面逐个回收 */
   block_idx = 0;
   while (block_idx < block_cnt) {
        if (all_blocks[block_idx] != 0) {
            block_bitmap_idx = 0;
            block_bitmap_idx = all_blocks[block_idx] - part->sb->data_start_lba;
            ASSERT(block_bitmap_idx > 0);
            byte_idx = block_bitmap_idx / 8;
            bit_odd  = block_bitmap_idx % 8;
            part->block_bitmap.bits[byte_idx] &= ~(1 << bit_odd);
            bitmap_sync(cur_part, block_bitmap_idx, BLOCK_BITMAP);
        }
        block_idx++; 
   }

/*2 回收该inode所占用的inode */
   byte_idx = inode_no / 8;
   bit_odd  = inode_no % 8;
   part->inode_bitmap.bits[byte_idx] &= ~(1 << bit_odd);
   bitmap_sync(cur_part, inode_no, INODE_BITMAP);

   /******     以下inode_delete是调试用的    ******
   * 此函数会在inode_table中将此inode清0,
   * 但实际上是不需要的,inode分配是由inode位图控制的,
   * 硬盘上的数据不需要清0,可以直接覆盖*/
   void* io_buf = vmm_malloc(1024,2);
   inode_delete(part, inode_no, io_buf);
   vmm_free(io_buf,1024);
   /***********************************************/
    
   inode_close(inode_to_del);
}
/* 初始化new_inode */
void inode_init(unsigned int inode_num,struct inode *new_inode){
    new_inode->i_num=inode_num;
    new_inode->i_size=0;
    new_inode->i_open_cnts=0;
    new_inode->write_lock=0;
    /* 初始化块索引数组i_sector */
    for(int i=0;i<13;i++){
        new_inode->i_sectors[i]=0;
    }
}
```



## inode.h

```
#ifndef _INODE_H_
#define _INODE_H_
#include "../stl/list.h"
#include "ide-dev.h"

/* inode结点 */
struct inode{
    unsigned int i_num; //inode编号
    /* 当此inode是文件，i_size表示文件大小
    当此inode是目录，i_size是指该目录下所有目录项大小之和 */
    unsigned int i_size; 

    unsigned int i_open_cnts; //记录文件被打开的次数
    char write_lock; //写文件时需要锁住，不能并行
    /* i_sectors[0-11]是直接数据块，i_sectors[12]用来存储一级间接块 */
    unsigned int i_sectors[13];
    //list_entry_t inode_tag;  //文件缓存列表
};
void inode_sync(struct partition *part,struct inode *inode,void *io_buf);
struct inode* inode_open(struct partition *part,unsigned int inode_no);
void inode_close(struct inode *inode);
void inode_init(unsigned int inode_num,struct inode *new_inode);
#endif
```



## super_block.h

```
#ifndef _SUPER_BLOCK_H_
#define _SUPER_BLOCK_H_

/* 
** 超级块-super block
*/
struct super_block{
    unsigned int magic; //魔数，用来标识文件系统
    unsigned int sec_cnt; //本分区总共扇区数
    unsigned int inode_cnt; //本分区中inode数量
    unsigned int part_lba_base; //本分区的起始lba地址

    unsigned int block_bitmap_lba; //块位图本身起始扇区地址
    unsigned int block_bitmap_sects; //块位图占用扇区数
    
    unsigned int inode_bitmap_lba; //inode节点位图本身起始扇区地址
    unsigned int inode_bitmap_sects; //inode节点位图占用扇区数

    unsigned int inode_table_lba; //inode表本身起始扇区地址
    unsigned int inode_table_sects; //inode表占用扇区数

    unsigned int data_start_lba; //数据块起始扇区地址
    unsigned int root_inode_no; //根目录所在I节点号
    unsigned int dir_entry_size; //根目录大小

    unsigned char pad[460]; //加上460字节，凑一个扇区
}__attribute__((packed));

#endif
```



#### 背景知识

整个目录包含了FreeFlyOS的文件系统实现，那么现在从上层到底层来分别介绍下这个文件系统吧！

呃，在此之前，先说明了这个文件系统的来源，主要借鉴了《操作系统真相还原》书中的ext2文件系统的实现，先大概看下其结构吧。

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-nAt3dRWI-1609681972036)(readme.assets/image-20210101195008006.png)\]](https://img-blog.csdnimg.cn/20210103215318927.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


大概就长上面这样，先看下我们这个OS所在硬盘的大小，如下图可以看到为5120000B（即10000个扇区）。

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-yEwbWp3C-1609681972038)(readme.assets/image-20210101195425185.png)\]](https://img-blog.csdnimg.cn/20210103215329458.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


然后说下这个硬盘的分区结构，呃，怎么说呢，在最初没有OS代码的时候，在Ubuntu环境下通过fdisk命令给这个虚拟硬盘进行了分区，只设置了一个主分区，就是从2048号-9999号扇区，有人可能会问为什么不从0号扇区开始呢？反正fdisk命令最低扇区号必须为2048，感兴趣的读者可以继续研究。

FreeFlyOS的硬盘分布如下所示:

0号扇区.                            =》bootblock(MBR)

1号扇区-473号扇区.         =》kernel

500号扇区-530号扇区      =》test_exec

530号扇区-570号扇区      =》test_cat

570号扇区-600号扇区      =》test_pipe

......

2048号扇区-9999号扇区  =》主分区

#### 主分区初始化

我们这里的主分区就是一个文件系统，和ext2文件系统类似，构造相同的结构。接下来一步一步地分析。

一个分区表项如下图所示，在使用fdisk创建主分区的时候，这些数据会自动存储在引导扇区（0号扇区）的0x1BE-0x1FD地址中，所以之前在写MBR的时候我们需要预留一定的空间给主分区表。

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-rHADOBIU-1609681972039)(readme.assets/image-20210101203558547.png)\]](https://img-blog.csdnimg.cn/20210103215344592.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


知道主分区的信息后，我们可以确定起始磁头号和结束磁头号，接着就可以布局我们的文件系统了。

操作系统引导块我们可以直接跳过，一般用于多系统启动，直接看超级块，也就是2048号扇区，其数据结构如下所示。

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-WExai0eL-1609681972040)(readme.assets/image-20210101211548668.png)\]](https://img-blog.csdnimg.cn/20210103215400120.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


由于我们人为设定了主分区最大文件数为1000，每个文件对应一个inode节点，所以可以计算出inode节点位图占用几个扇区，此外ino de节点如下图所示。

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-acvgv1BB-1609681972041)(readme.assets/image-20210101212332720.png)\]](https://img-blog.csdnimg.cn/20210103215414675.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


那么inode节点大小确定了，就可以确定inode数组所占扇区，接着我们就可以计算剩余的空间能留下多少个空闲块位图和空闲块了，第0号空闲块为根目录所在区域，这样就确定了超级块的内容，直接写到2048号扇区中保存。

接着我们需要确定根目录了，首先给空闲块位图起始扇区中的第0位置1，表示该块已占用，由于根目录对应一个inode节点，接着把inode位图的第0位置1，表示该inode节点已被使用，其对应的实体也就是inode节点数组的第0项进行根目录设置，最后对空闲块起始扇区的根目录项进行设置，如下图所示，这样根目录就完成了。

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-DptjlKbX-1609681972041)(readme.assets/image-20210101214644423.png)\]](https://img-blog.csdnimg.cn/20210103215428573.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


为了使用方便，我们在文件系统初始化的时候，把下图的信息先读到内存中。

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-rcqdxP4y-1609681972042)(readme.assets/image-20210101214931418.png)\]](https://img-blog.csdnimg.cn/20210103215440928.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


同样，把下图所示的根目录结构先读到内存中，具体步骤是先从主分区表项中获取超级块的根目录所在i节点号，然后将i节点数组中对应的节点读到根目录中。

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-YRjL2VIm-1609681972043)(readme.assets/image-20210101215216128.png)\]](https://img-blog.csdnimg.cn/20210103215452342.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


最后初始化文件表结构，如下图，全部清0。到此为止，分区信息已经初始化完毕。

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-eWxpwYDY-1609681972044)(readme.assets/image-20210101220317718.png)\]](https://img-blog.csdnimg.cn/2021010321550393.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


#### 打开文件

FreeFlyOS中打开文件函数为：int sys_open(const char* pathname, unsigned char flags)，和我们平时用的open有点类似。

首先输入要创建的文件路径，然后我们会对这个路径进行一个简单的解析，比如/a/b/c，

过程如下：

1、先算出这个文件路径的目录深度，该例为3(a、b、c共3层)。

2、然后在所给的目录中查找有无相同名字的文件，首先解析出a目录是否在根目录/下，读出根目录i节点对应的数据块信息，其中包含了根目录下的所有目录项，然后依次比对目录名称，若找到，则返回对应的目录项信息，读取目录项对应的i节点信息，然后返回目录信息，继续往下查找，当找到最后一层文件时，返回该文件的i节点号，若没找到则返回-1。

3、如果是在最后一个路径上没找到，则通过flags标志判断是不是要创建文件，若需要创建文件，则继续第4步，若是打开已存在文件，则继续第5步。

4、先分配一个i节点号，然后从全局文件描述符表中获取一个空闲位的下标，然后将文件表空闲位对应的文件指向新分配的i节点，然后将该文件目录项安装在父目录的下面，然后将父目录i节点数据同步到硬盘，接着将新创建文件的i节点同步到硬盘，同样将inode位图的信息更新到硬盘中，最后将全局文件描述符表对应信息安装到当前进程自己的文件描述符表中。

5、打开相应i节点号对应的inode节点信息，并安装到全局文件描述符表中，然后还需要判断下如果flags标志包含写文件，则需要判断有没有其他进程正在写该文件，最后将对应的文件信息安装到当前进程自己的文件描述符表中。

总结，打开文件最后返回的是当前进程文件描述符表的下标，通过该下标可以找到全局文件描述符表的下标，从而找到对应的文件描述符，包含文件偏移量，文件标志以及文件i节点信息。

#### 关闭文件

FreeFlyOS中关闭文件函数为：int sys_close(int fd)。

简单来讲，就是释放文件描述符对应信息的内存等。

#### 写文件

FreeFlyOS中写文件函数为：int sys_write(int fd, const void* buf, unsigned int count)。

1、首先我们知道当前进程下的文件描述符，然后通过它获取全局文件描述符，从而获得对应的i节点。

2、首先判断文件是否第一次写，如果是，先分配一个空闲块，然后同步下空闲块位图信息，然后分为3种情况，将数据写到i节点对应的数据块中，一种是12个直接数据块够用；一种是数据需要写到一级间接块表中，此时新创建一级间接块表；最后一种是之前的数据已经包含了一级间接块表，此时新建间接块表项即可。

3、同步到硬盘中。

#### 读文件

FreeFlyOS中读文件函数为：int sys_read(int fd, void* buf, unsigned int count)

1、首先我们知道当前进程下的文件描述符，然后通过它获取全局文件描述符，从而获得对应的i节点。

2、和写文件一样，读取i节点对应的全部数据块，将其读入到buf中。

#### 删除文件

FreeFlyOS中删除文件函数为：int sys_unlink(const char* pathname) 

1、删除父目录的对应目录项。

2、删除文件对应的i节点等信息。

3、同步到硬盘。

#### 创建目录

FreeFlyOS中创建目录函数为： int sys_mkdir(const char* pathname)

1、将自己的目录项装在父目录的目录项中。

2、在自己的目录项下装载.和..目录项 。

3、数据同步到硬盘中。

#### 删除目录

FreeFlyOS中删除目录函数为：int sys_rmdir(const char* pathname)

判断该目录下的目录项是否为空，若为空，则删除父目录下对应的目录项。

#### 更改当前工作目录

FreeFlyOS中更改当前工作目录函数为：int sys_chdir(const char* path)

在根目录下逐步寻找目录项，一层一层往下递进，若找到对应的目录项，则把其i节点号放在当前进程中。

#### 获取当前工作目录

FreeFlyOS中获取当前路径函数为：char* sys_getcwd(char* buf, unsigned int size)

通过当前进程获取当前目录的i节点信息，然后逐层往上寻找目录项，直到根目录为止，并把其绝对地址写入buf中。



大概就说这么多吧，整个文件系统算是比较简单的一种，直接在内存中进行读取，还没有涉及缓冲区等信息。
