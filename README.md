# STM32_IAP
----
实现STM32通过IAP进行程序更新升级

不会编辑GitHub...

## 一、代码分区烧录
----
创建3个STM32工程：bootloader、user1、uesr2

                  烧录地址        size     
    bootloader    0x0800 0000    0x4000   16k
    user1         0x0800 4000    0x5c00   32k
    user2         0x0800 9c00    0x5c00   32k
    
### boot.c

存放在bootloader项目工程下

跳转函数

    void Load_app(uint32_t appaddr)
    {
      if(((*(vu32*)app_addr)&0x2FFE0000)==0x20000000){
        JumpToApp = (IapFun)*(vu32*)(app_addr+4);
        MSR_MSP(*(vu32*)app_addr);
        JumpToApp();
      }
    }

