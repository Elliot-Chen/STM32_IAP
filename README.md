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
    
### （1） bootloader工程

该工程下包含的用户文件：

    boot.c
    bsp_usart1.c
    bsp_gpio.c

跳转函数：Load_app

```c
 void Load_app(uint32_t appaddr)
{
  if(((*(vu32*)app_addr)&0x2FFE0000)==0x20000000){
    JumpToApp = (IapFun)*(vu32*)(app_addr+4);
    MSR_MSP(*(vu32*)app_addr);
    JumpToApp();
  }
}
```
