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
由API的参数可以知道，对flash操作时，最小单位位2byte，即所谓的半字

### （二）准备工作
建立bsp_flash.c和bsp_flash.h文件，其中在头文件中定义必要的变量
```c
/*
	一个扇区为1024byte(看具体芯片数据手册)
	u8 arr[1024] size = 1k byte
	u16 arr[512] size = 1k byte
	u32 arr[256] size = 1k byte
*/

#define STM32_FLASH_SIZE	64			//芯片flash大小，单位Kbyte

#if STM32_FLASH_SIZE < 256				//扇区大小，单位byte
#define SECTOR_SIZE 1024
#else
#define SECTOR_SIZE 2048
#endif

extern uint16_t FLASH_BUF[SECTOR_SIZE/2];			//写入falsh专用BUFF，size = 扇区大小/2
```
测试使用的芯片型号是stm32f103c8t6，64k flash和20k sram

FLASH_BUF数组由于flash操作过程中数据的临时存放位置

### （三）flash数据读出函数

相对于flash写入，flash读出简单得多

```c
/**
  * @brief  在指定地址读出数据，读半字(2byte)
  * @param  address：	指定的地址
  * @retval 		返回指定地址的数据
			类型uint16_t
  */
uint16_t FLASH_ReadHalfWord(uint32_t address)
{
	return *(vu16*)address;
}

/**
  * @brief  在指定地址读出数据，读全字(4byte)
  * @param  address：	指定的地址
  * @retval 		返回指定地址的数据
			类型uint32_t
  */
uint32_t FLASH_ReadWord(uint32_t address)
{
	uint32_t temp1;	
	
	temp1 = *(vu16*)(address+2);
	temp1 = temp1 << 16;
	temp1 += *(vu16*)address;
	
	return temp1;
}

/**
  * @brief  读出指定地址往后的指定个数的数据
  * @param  DATA_Address：	指定的地址
  * @param  *pDATA：		存放读出数据的数组
  * @param  DATA_NUM:		读出的个数（半字）
				值(0 ~ SECTOR_SIZE/2)
  * @retval none
  */
void FLASH_ReadmoreData(uint32_t DATA_Address, uint16_t *pDATA, uint32_t DATA_NUM)
{
	u32 i;
	for(i = 0; i < DATA_NUM; i++){
		pDATA[i] = FLASH_ReadHalfWord(DATA_Address + i*2);
	}
}
```
第三个函数使用前需要先定义一个用于接收数据的数组，最终读出的数据存放在该数组中

### （四）flash数据写入函数

flash数据写入流程：
	
	1、解锁
	2、读出扇区内容（如果需要）
	3、对整个扇区进行擦除，因为规定flash在写入前初始值必须为0xFF
	4、调用API进行写操作
	5、重新上锁
	
根据以上的流程，可以定制专属的flash写入函数

不过需要注意的是，flash写入前要求所写地址内的值为 0xFF ，否则不能写入，也就是说如果之前存在数据而且不是0xFF，必须要先调用FLASH_ErasePage()进行擦除，而该函数会擦除掉整个扇区，因此在这里选择直接读出扇区所有数据后再整块擦除，写入数据。

---
```c
/**
  * @brief  在指定扇区写入指定数量的数据
  * @param  DATA_Address：	指定的扇区首地址
  * @param  *pDATA：		指针，指向存储数据的数组
  * @param  DATA_NUM:		传入数组的大小（数据个数，单个数据大小2byte）
				值(0 ~ SECTOR_SIZE/2)
  * @retval none
  */
void FLASH_WriteSector(uint32_t DATA_Address, uint16_t *pDATA, uint32_t DATA_NUM)
{
	uint32_t dataIndex;
	//检查地址是否合法
	if(DATA_Address < FLASH_BASE || ((DATA_Address + DATA_NUM*2) >= (FLASH_BASE + 1024*STM32_FLASH_SIZE))){
		return;
	}
	else{
		//解锁flash
		FLASH_Unlock();
		//擦除扇区
		FLASH_ErasePage(DATA_Address);
		//写入数据
		for(dataIndex = 0; dataIndex < DATA_NUM; dataIndex++){
			FLASH_ProgramHalfWord(DATA_Address + dataIndex*2, pDATA[dataIndex]);
		}
		//锁定flash
		FLASH_Lock();	
	}
}
```
▲ 这个函数实现了不保留的数据写入，最大的数据长度为扇区长度，值得注意的数DATA_NUM参数，由于单个数据大小为2byte，所以该值取值范围是0~511,共512个数据，最终的写入数据大小size = 512 * 2 byte = 1024 byte 即扇区size

参数DATA_Address必须为扇区首地址

---
```c
/**
  * @brief  在指定地址写入2byte数据（半字），不破坏前后数据
  * @param  DATA_Address：	指定的地址
  * @param  *pDATA：		数组指针，用于临时存储数据的数组
  * @param  DATA:		传入数组的大小（数组的个数）
				值(0 ~ SECTOR_SIZE/2)
  * @retval none
  */
void Flash_Write16Bit(uint32_t DATA_Address, uint16_t *pDATA, uint16_t DATA)
{
	uint32_t secaddr;		//扇区相对偏移量（相对基地址）
	uint32_t secnum;		//扇区编号
	uint32_t secoff;		//扇区内偏移量，由此可知该地址在该扇区内前面有多少个数据
	uint32_t framware_addr;		//传入地址所在扇区的扇区物理绝对地址
	uint32_t dataIndex = 0;
	uint32_t i = 0;
		
	secaddr = DATA_Address - FLASH_BASE;			//计算扇区偏移量
	secnum = secaddr / SECTOR_SIZE;				//扇区编号 = 相对地址 / 扇区大小
	framware_addr = secnum * SECTOR_SIZE + FLASH_BASE;	//传入地址所在的扇区的首地址
	secoff = secaddr % SECTOR_SIZE;				//扇区内偏移量，即插入数据在该扇区内的位置
	
	//读出整个扇区数据
	FLASH_ReadmoreData(framware_addr, pDATA, SECTOR_SIZE/2);
	//替换需要写入的数据
	FLASH_BUF[secoff/2] = DATA;
	//写入整个扇区数据
	FLASH_WriteSector(framware_addr, pDATA, SECTOR_SIZE/2);
}
```
▲ 这个函数结合了FLASH_ReadmoreData()和FLASH_WriteSector()两个函数，实现了在特定地址写入2byte数据，不损坏前后的数据

---
以上构建的两个flash写入函数，直接调用就可以往flash里写入数据了

其中FLASH_WriteSector()是接下来实现IAP功能所需要的函数，不保留写入

而Flash_Write16Bit()是用于各类标识符的写入

根据给出的flash写入流程，也可以构建更多功能的写入函数，这里不再累述

## 三、	IAP实现

通过上面两个测试，程序间的跳转和flash写入都已经单独实现了，对两部分进行结合即可实现IAP功能

首先定义一些必要的变量和声明
```c
#define boot_addr		0x8000000		//boot地址
#define APP1_addr		0x8004000		//APP1地址
#define	APP2_addr		0x8009c00		//APP2地址

#define Updata_flag		0x800f800		//更新标志位，0不需要更新，1执行更新程序
#define newAPP_flag		0x800f802		//新APP标志位，选择使用哪个APP 0 ~ 255
#define oldAPP_flag		0x800f804		//旧APP标志位，选择使用哪个APP 0 ~ 255
#define SIZE_flag		0x800f806		//接收到的APP size检验位  类型u16

extern uint32_t SIZE;				//接收到的APP固件大小，单位byte

/*先改变数组内的值，再写入flash*/
#define flag_num		20			//AT指令个数
extern uint16_t flag_buf[flag_num];				//flag临时存储BUFF，用于存储AT指令
```
开头三个地址是对第一部分跳转测试中的拓展，boot地址不变，分配16Kflash

然后划分两个APP区间，每个分配23Kflash,每次只有一个APP处在运行状态，另一个处于空闲状态，下一次更新APP时写进空闲的APP，然后再该地址上运行新的APP，原先的APP则空闲出来，实现两个APP轮流使用，若更新失败，还能回滚。

从0x800 f800--0x801 0000这2K是保留空间，用于存储各类flag标志位

SIZE用于计算串口接收到的数据个数，在串口中断服务程序中实现，单个数据大小为1byte












