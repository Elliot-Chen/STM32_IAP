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

### （一）代码分区烧录

创建2个STM32工程：bootloader、app

配置不同工程的烧录地址(keil MDK)

在工程配置界面Options for Targer 'xxx'的Target选项中填写起始地址和大小（每个工程的空间可自由设置）

            start          size     
    boot    0x0800 0000    0x4000   16k
    app     0x0800 4000    0x5c00   32k
    
### （1） boot工程

该工程下包含的用户文件：

    boot.c
    bsp_usart1.c
    bsp_gpio.c

boot.c文件关键内容如下

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


//函数：    Load_app
//功能：    实现不同工程之间的跳转
//参数：    appaddr    工程起始地址
//返回：    无
void Load_app(uint32_t appaddr)
{
  if(((*(vu32*)app_addr)&0x2FFE0000)==0x20000000){
    JumpToApp = (IapFun)*(vu32*)(app_addr+4);
    MSR_MSP(*(vu32*)app_addr);
    JumpToApp();
  }
}
```

该函数通过地址来实现不同工程之间的跳转。

主函数如下

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

显然在boot主程序中初始化外设配置后，仅仅调用了一个Load_app函数。








