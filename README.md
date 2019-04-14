# STM32_IAP

实现STM32通过IAP进行程序更新升级

不会编辑GitHub...

开发环境：keil5

芯片：stm32f103c8t6

flash size : 64k

SRAM : 20k

使用3.5库开发

例程代码等实现功能后会上传

不定期更新

## 一、跳转测试

要实现IAP功能，其中关键是实现程序之间的跳转

芯片上电后检测是否需要更新，若不需要则从boot跳转到user程序，在user程序下如果检测到需要更新的标志位，则跳转到boot下进行更新。

下面详细说明如何实现不同程序之间的跳转

### （一） 代码分区烧录

创建2个STM32工程：bootloader、app

配置不同工程的烧录地址(keil MDK)

在工程配置界面Options for Targer 'xxx'的Target选项中填写起始地址和大小（每个工程的空间可自由设置）

            start          size     
    boot    0x0800 0000    0x4000   16k
    app     0x0800 4000    0x5C00   32k
    
### （二） boot工程

该工程下包含的用户文件：

    boot.c
    bsp_usart1.c
    bsp_gpio.c

boot.c文件关键内容 ▼
---
```c
typedef  void (*IapFun)(void);
IapFun JumpToApp; 

__asm void MSR_MSP(uint32_t addr) 
{
    MSR MSP, r0
    BX r14
//  __ASM("msr msp, r0");
//  __ASM("bx lr"); 
}


//函数：    Load_addr
//功能：    实现不同工程之间的跳转
//参数：    newaddr    工程起始地址
//返回：    无
void Load_addr(uint32_t newaddr)
{
    if(((*(vu32*)newaddr)&0x2FFE0000) == 0x20000000){
	JumpToAddr = (IapFun)*(vu32*)(newaddr + 4);
	MSR_MSP(*(vu32*)newaddr);
	JumpToAddr();
    }
    else{
	Printf("no app");
	delay_ms(500);
	Printf("return boot");
	delay_ms(500);
    }
}
```

Load_addr函数分析：

---
	if(((*(vu32*)newaddr)&0x2FFE0000) == 0x20000000)

newaddr是用户程序Flash首地址

(*(vu32*)newaddr)意思是取用户程序首地址里面的数据，该数据是用户程序的堆栈地址，堆栈地址指向RAM，RAM起始地址为0x2000 0000（也可以自己设定）

该判断语句实现了：判断用户代码的堆栈地址是否落在0x2000 0000 ~0x2001 FFFF区间

若堆栈不合法，则执行else里面的内容

---
	JumpToAddr = (IapFun)*(vu32*)(newaddr+4);

stm32复位后会先从（堆栈地址+4）中取出复位中断向量的地址，并且跳转到中断服务程序，中断服务程序执行完后再跳转到main函数。

JumpToAddr取出的的正是复位中断向量地址

---
	MSR_MSP(*(vu32*)newaddr);
	
这里调用了函数__asm void MSR_MSP(uint32_t addr)，用于设置用户程序堆栈指针	

由于涉及到C语言中的融合汇编的编程，以后有机会再补充说明

---
	JumpToAddr();
	
这是一个指针函数，用于执行复位中断

主函数 ▼
---
```c
int main(void)
{
    USART1_Config();
    NVIC_Configuration();	
    LED_GPIO_Config();
    Printf("Entering boot");
	
    Updata_app_flag = 0;
	
    while(1){
        if(Updata_app_flag){
            Printf("Updata app");
            delay_ms(500);
            Updata_app();
         }
         else{
            Printf("no updata signal");
            Printf("Loading app");
            delay_ms(500);
            Load_app(app_addr);
         }
     }
}
```

在while循环体，关于app升级在后面会单独讲解，所以第一个if判断可以先忽略

显然在boot主程序中初始化外设配置后，仅仅调用了一个Load_app函数，通过传递用户程序地址即可实现不同的工程之间跳转功能。








