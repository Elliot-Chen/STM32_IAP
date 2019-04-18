# STM32_IAP

实现STM32通过IAP进行程序更新升级

不会编辑GitHub...

开发环境：keil5

芯片：stm32f103c8t6

flash size : 64k

SRAM : 20k

使用3.5库开发

使用st-link v2进行烧写

例程代码等实现功能后会上传

不定期更新

## 一、跳转测试

要实现IAP功能，其中关键之一是实现程序之间的跳转

芯片上电后检测是否需要更新，若不需要则从boot跳转到user程序，在user程序下如果检测到需要更新的标志位，则跳转到boot下进行更新。

下面详细说明如何实现不同程序之间的跳转。

### （一） 代码分区烧录

创建2个STM32工程：bootloader、app

配置不同工程的烧录地址(keil MDK)

在工程配置界面Options for Targer 'xxx'的Target中填写起始地址和大小（每个工程的分配空间可自由设置）

    IROM1   start          size     
    boot    0x0800 0000    0x4000   16k
    app     0x0800 4000    0x5C00   23k
    
在工程配置界面Options for Targer 'xxx'的Debug中选择ST-Link Debugger，并且点击旁边的setting按钮

在弹出的窗口中点击Flash Download，可以看到下面信息 ▼ 


	Erase Full Chip		Program
	Erase Sectors		Verify
	Do not Erase		Reset and Run
	
选择Erase Full Chip会擦除整个flash

本次调试中boot程序和app程序是通过st-link分两次进行烧写，因此要选择Erase Sectors擦除用到的扇区	
    
### （二）地址偏移

在芯片上电启动时，会首先调用systemInit函数初始化时钟，同时还会进行中断向量表的设置

在systemInit函数体结尾处可以看到

```c
#ifdef VECT_TAB_SRAM
  SCB->VTOR = SRAM_BASE | VECT_TAB_OFFSET; /* Vector Table Relocation in Internal SRAM. */
#else
  SCB->VTOR = FLASH_BASE | VECT_TAB_OFFSET; /* Vector Table Relocation in Internal FLASH. */
#endif 
```
VTOR寄存器存放的是中断向量表的起始地址，默认情况下VECT_TAB_SRAM是没用定义的，

所以执行SCB->VTOR = FLASH_BASE | VECT_TAB_OFFSET;

追踪变量可以得知

	#define FLASH_BASE            ((uint32_t)0x08000000) /*!< FLASH base address in the alias region */
	#define SRAM_BASE             ((uint32_t)0x20000000) /*!< SRAM base address in the alias region */
	
	#define VECT_TAB_OFFSET  0x0 /*!< Vector Table base offset field. 
                                  This value must be a multiple of 0x200. */

其中FLASH_BASE和SRAM_BASE都是固定值（可以在对应芯片的数据手册上的Memory mapping章节找到）

而VECT_TAB_OFFSET是默认值0，该值表示中断向量表起始地址的偏移量

对于boot工程，偏移量为0，不用修改

对于app工程，偏移量为0x4000，上述代码则应修改为SCB->VTOR = FLASH_BASE | 0x4000; 当然也可以选择改变VECT_TAB_OFFSET的值

### （三） 跳转函数

创建用户文件boot.c，以库形式进行开发测试，便于移植

该文件中存放了实现不同工程之间跳转的必要函数 ▼

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

JumpToAddr取出的正是复位中断向量地址

---
	MSR_MSP(*(vu32*)newaddr);
	
这里调用了函数__asm void MSR_MSP(uint32_t addr)，用于设置用户程序堆栈指针	

由于涉及到C语言中的融合汇编的编程方式，以后有机会补习后再补充说明

---
	JumpToAddr();
	
这是一个指针函数，用于执行复位中断

---


### （四） 主函数

将boot.c分别放入boot和app工程中，调用Load_addr即可实现程序之间的跳转

下面以boot工程的主函数为例说明
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
            Load_addr(app_addr);
         }
     }
}
```

在while循环体中，关于app升级在后面会单独讲解，所以第一个if判断可以先忽略

显然在主程序中初始化外设配置后，仅仅调用了一个Load_app(app_addr)函数，通过传递app程序地址即可实现boot跳转到app工程。

在boot.h中有宏定义，该地址就是（一）中设定的起始地址

	#define boot_addr	0x8000000
	#define app_addr	0x8004000

在app工程中以库形式加入boot.c和boot.h文件，调用Load_addr(boot_addr)函数也可以实现从app跳转到boot工程。

## 二、Flash数据存储测试

实现IAP功能的另一关键在于对Flash扇区的数据存储操作。

芯片在上电运行过程中通过串口接收需要更新的bin文件，可以临时存储在SRAM中，掉电不保存，需要更新的固件必须要写入Flash中

程序运行流程：

	芯片上电 → 运行boot程序 → 判断标志位（是否需要升级固件）
					↓		↓
			↑	       需要		不需要
					↓		↓
			↑	  执行更新函数		运行app程序
					↓		↓
			↑	←  清除标志位		主循环	← 产生标志位 ← 串口传输新固件
							↓（检测标志位）
			↑	←	←	←跳转到boot程序

建立bsp_flash.c和bsp_flash.h文件
### （一）ST官方flash库文件

芯片在上电运行中一般是不允许进行flash写入操作的，但ST官方提供了针对flsah操作的API接口。

在stm32f10x_flash.h文件中可以看到所提供的API

其中需要关注的是

```c
void FLASH_Unlock(void);//flash解锁，在写flash前必须要先解锁
void FLASH_Lock(void);	//flsah锁定，在写入flash后必须要上锁

FLASH_Status FLASH_ErasePage(uint32_t Page_Address);//扇区擦除

FLASH_Status FLASH_ProgramWord(uint32_t Address, uint32_t Data);	//写全字（32bit即4byte）
FLASH_Status FLASH_ProgramHalfWord(uint32_t Address, uint16_t Data);	//写半字（16bit即2byte）

```
flash写入流程：
	
	1、解锁
	2、读出扇区内容（如果需要）
	3、对整个扇区进行擦除，因为规定flash在写入前初始值必须为0xFF
	4、调用API进行写操作
	5、重新上锁



