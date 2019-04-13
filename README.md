# STM32_IAP
实现STM32通过IAP进行程序更新升级

不会编辑GitHub...

## 一、代码分区烧录
创建3个STM32工程：bootloader user1 uesr2
              烧录地址        size
bootloader    0x0800 0000    0x4000   16k
user1         0x0800 4000    0x5c00   32k
user2         0x0800 9c00    0x5c00   32k
