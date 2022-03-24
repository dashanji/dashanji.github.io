---
title: Chapter-5 VGA部分详解
description: Chapter 5 of FreeFlyOS
toc: true
authors:
tags:
weight: 5
categories:
series:
date: '2022-03-24'
lastmod: '2022-03-24'
draft: false
---
## vga.c

```
#include "vga.h"
#include "../asm/asm.h"
#include "../serial/serial.h"
#include "../keyboard/keyboard.h"
#define NULL ((void *)0)
//the standard window is 80*25, max of cursor_x is 25, max of cursor_y is 80
static unsigned char cursor_x=0;
static unsigned char cursor_y=0;
//the start address of CGA is 0xB8000，此时开启了分页，应加上一个线性映射地址
//避免后面新建页表时，未映射前面一段物理内存而导致BUG
static unsigned short *vga_memory=(unsigned short *)(0xB8000+0xC0000000);
/* print cursor */ 
inline void print_cursor(unsigned char x,unsigned char y){
    unsigned short pos;
    //calculate the pos from (0,0)
    pos=x*80+y;

    //write register data to port 0x3d4,set register_cursor_x
    outb(VGA_register,register_cursor_x);
    //set cursor_x
    outb(VGA_data,(pos>>8)&0xFF);
    //write register data to port 0x3d4,set register_cursor_y
    outb(VGA_register,register_cursor_y);
    //set cursor_y
    outb(VGA_data,pos&0xFF);
}
/* use 'space' to clear the screen */
inline void clear(){
    unsigned short attribute=((0<<4)&0xf0|(15&0xf))<<8;
    unsigned short space=' '|attribute;
    for(unsigned short i=0;i<25*80;i++){
        vga_memory[i]=space;
    }

    cursor_x=0;
    cursor_y=0;

    print_cursor(cursor_x,cursor_y);
}
/* print char */ 
inline void print_char(char c,color_type background,color_type foreground){
    unsigned short attribute=0,character=0,pos;
   
    if(cursor_x>=25){
        clear();
    }
        
    //calculate the pos from (0,0)
    pos=cursor_x*80+cursor_y;

    switch(c)
    {
        case '\n':
            cursor_y=0;
            cursor_x++;
            break;
        case '\t':
            cursor_y=(cursor_y+8)&~(8-1);
            break;
        default:
            //set the first byte of character
            character=c;
            //set the last byte of character
            attribute=((background<<4)&0xf0|(foreground&0xf))<<8;
            character=character|attribute;
            //send character to vga memory
            vga_memory[pos]=character;

            cursor_y++;
            if(cursor_y>=80)
            {
                cursor_y=0;
                cursor_x++;
            }
            break;
    }
    print_cursor(cursor_x,cursor_y);
}
//print string
inline void print_string(char *str,color_type background,color_type foreground){
    //the end of string is '\0' and its ascii is 0
    while(*str){
        print_char(*str++,background,foreground);
    }
}
/*                print num                                 
*  base-represent the displayed type is hex or dec: 
*      base=10-----hex base=16-----dec
*       flag-1:display len bits num
*       flag-0:dislay only num
*/
inline void print_num(unsigned long long num,
color_type background,color_type foreground,unsigned char base,char len,int flag){
    
    char str[20];
    char i,j,k;
    str[len]='\0';

    for(i=len-1;i>=0;i--){
        str[i]=do_div(num,base);
        if(base==10){
            str[i]=str[i]+'0';
        }  
        else if(base==16){
            str[i]=str[i]>9?str[i]-10+'A':str[i]+'0';
        }   
    }

    if(!flag){
        int x=0;
        while(str[x++]=='0');
        x--;
        if(str[x]!=NULL)
            print_string(&str[x],background,foreground);
        else
            print_char('0',background,foreground);
    
        return ;
    }

    print_string(str,background,foreground);
}

/* 退格 */
void backtrace(){
    if(cursor_y>21){
    cursor_y--;
    unsigned short attribute=0,character=0,pos;
   
    if(cursor_x>=25){
        clear();
    }
        
    //calculate the pos from (0,0)
    pos=cursor_x*80+cursor_y;

    //set the first byte of character
    character=' ';
    //set the last byte of character
    attribute=((default_background<<4)&0xf0|(default_foreground&0xf))<<8;
    character=character|attribute;
    //send character to vga memory
    vga_memory[pos]=character;

    print_cursor(cursor_x,cursor_y);
    }
}
/*
*    printk(“”,...)-standardized format output
*         %c-------char
*         %s-------char *
*         %d-------dec int
*      %x,%X-------hex int  
*        %ud-------unsigned dec int (10)
*    %ux,%uX-------unsigned hex int (16)
*         %l-------hex long long
*        %ul-------unsigned hex long long
*/
void printk(char *fmt,...){
    va_list ap;
    
    char c;
    char *str;

    int dec_num;
    int hex_num;

    unsigned int unsigned_dec_num;
    unsigned int unsigned_hex_num;

    long long ll_hex_num;

    unsigned long long ull_hex_num;

    char bits=0;     //record the number's bits

    va_start(ap,fmt);

    while(*fmt){
        if(*fmt=='%'){
dis_num:    switch (*(++fmt))
            {
                case 'c':
                    c=va_arg(ap,char);
                    print_char(c,default_background,default_foreground);
                    break;
                case 's':
                    str=va_arg(ap,char *);
                    print_string(str,default_background,default_foreground);
                    break;
                case 'd':
                    dec_num=va_arg(ap,int);
                    if(bits){
                        print_num(dec_num,default_background,default_foreground,dec,bits,display_bits);
                    }
                    else{
                        print_num(dec_num,default_background,default_foreground,dec,ulonglong_max,display_num);
                    }
                    break;
                case 'X':
                case 'x':
                    hex_num=va_arg(ap,int);
                    if(bits){
                        print_num(hex_num,default_background,default_foreground,hex,bits,display_bits);
                    }
                    else{
                        print_num(hex_num,default_background,default_foreground,hex,ulonglong_max,display_num);
                    }
                    break;
                case 'l':
                case 'L':
                    ll_hex_num=va_arg(ap,long long);
                    if(bits){
                        print_num(ll_hex_num,default_background,default_foreground,hex,bits,display_bits);
                    }
                    else{
                        print_num(ll_hex_num,default_background,default_foreground,hex,ulonglong_max,display_num);
                    }
                    break;
                case 'u':
                    switch (*(++fmt))
                    {
                        case 'd':
                            unsigned_dec_num=va_arg(ap,unsigned int);
                            if(bits){
                                print_num(unsigned_dec_num,default_background,default_foreground,dec,bits,display_bits);
                            }
                            else{
                                print_num(unsigned_dec_num,default_background,default_foreground,dec,ulonglong_max,display_num);
                            }
                            break;
                        case 'X':
                        case 'x':
                            unsigned_hex_num=va_arg(ap,unsigned int);
                            if(bits){
                                print_num(unsigned_hex_num,default_background,default_foreground,hex,bits,display_bits);
                            }
                            else{
                                print_num(unsigned_hex_num,default_background,default_foreground,hex,ulonglong_max,display_num);
                            }
                            break;
                        case 'l':
                        case 'L':
                            ull_hex_num=va_arg(ap,unsigned long long);
                            if(bits){
                                print_num(ull_hex_num,default_background,default_foreground,hex,bits,display_bits);
                            }
                            else{
                                print_num(ull_hex_num,default_background,default_foreground,hex,ulonglong_max,display_num);
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
                    goto dis_num;
                    break;
                default:    
                    print_string("error format!Please correct it!",default_background,default_foreground);
                    break;
            }
            fmt++;
        }
        else{
            print_char(*fmt,default_background,default_foreground);
            fmt++;
        }
        bits=0;
    }
}

/* 
*  cons_putc - print a single character @c to console devices 
*/
void cons_putc(int c) {

        lpt_putc(c);
        if(c){
            printk("%c",c);
        }
        serial_putc(c);
}

int cons_getc(void) {
    int c = 0;

    // poll for any pending input characters,
    // so that this function works even when interrupts are disabled
    // (e.g., when called from the kernel monitor).
    serial_intr();
    kbd_intr();
    
    // grab the next character from the input buffer.
    if (cons.rpos != cons.wpos) {
        c = cons.buf[cons.rpos ++];
        if (cons.rpos == CONSBUFSIZE) {
            cons.rpos = 0;
        }
        //return c;
    }
    return c;
}
```

#### Vga.h

```
#ifndef _VGA_H_
#define _VGA_H_

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

#define VGA_register 0x3d4
#define VGA_data 0x3d5
#define register_cursor_x 14
#define register_cursor_y 15

/*可变参数宏定义*/
typedef char * va_list;     
//为了满足需要内存对齐的系统
#define _INTSIZEOF(n)  ((sizeof(n)+sizeof(int)-1)&~(sizeof(int) - 1) ) 

/*
*  va_start(ap,v):ap指向第一个变参的位置，即将第一个变参的地址赋予ap
*  va_arg(ap,t):获取变参的具体内容，t为变参的类型，如有多个参数，
*               则通过移动ap的指针来获得变参的地址，从而获得内容
*  va_end(ap):清空va_list，即结束变参的获取,防止野指针
*/
#define va_start(ap,v) ( ap = (va_list)&v + _INTSIZEOF(v) )     
#define va_arg(ap,t)   ( *(t *)((ap += _INTSIZEOF(t)) - _INTSIZEOF(t)) )   
#define va_end(ap)     ( ap = (va_list)0 )   

/* 系统默认的背景色和字体颜色，可根据喜好更改 */
#define default_background black
#define default_foreground green

typedef enum color {
    black = 0,
    blue = 1,
    green = 2,
    cyan = 3,
    red = 4,
    magenta = 5,
    brown = 6,
    light_grey = 7,
    dark_grey = 8,
    light_blue = 9,
    light_green = 10,
    light_cyan = 11,
    light_red = 12,
    light_magenta = 13,
    light_brown  = 14,   // yellow
    white = 15
}color_type;

void print_cursor(unsigned char x,unsigned char y);
void print_char(char c,color_type background,color_type foreground);
void clear();
void print_string(char *str,color_type background,color_type foreground);
void print_num(unsigned long long num,color_type background,color_type foreground,unsigned char base,char len,int flag);
void printk(char *fmt,...);
void cons_putc(int c);
int cons_getc(void);
void backtrace();
#endif
```

vga即显示屏幕初始化，包含清空屏幕，打印字符，从键盘和串口接收字符等。

关于VGA显示的问题，详见我的博客https://blog.csdn.net/cy295957410/article/details/108436730
