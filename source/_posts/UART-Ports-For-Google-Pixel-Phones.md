---
title: UART Ports For Google Pixel Phones
date: 2024-08-12 21:06:57
tags:
---

# Pixel 手机的 UART 针脚定义
## 启用 UART 终端
在 Ｐixel 上启用 UART 需要已解锁的 bootloader
解锁后，进入 bootloader，执行
`fastboot oem uart enable`
执行结果返回“OKAY”即开启 UART 终端
UART参数为115200n8，没有硬件串口流控制
## 对于存在3.5mm耳机口的型号
UART从3.5mm接口引出，TX电压应设为1.8V（3.3V应该不会有问题），Sleeve施加3.3V不会有问题
若UART适配器只支持3.3V，可在TXD和GND加2k和2.4k电阻分压
TRRS定义为：将TRRS接头的尖头朝左放置，从左到右按塑料环为分割，分别是Tip、Ring1、Ring2、Sleeve
UART接线定义如下：

UART | TRRS
--|:--
3.3V | Sleeve
GND | Ring2
TXD | Ring1
RXD | Tip

## 对于仅有 USB-C 而没有3.5mm的型号
UART从USB-C中引出
USB-C线材必须引出USB2.0, SBU1/SBU2, CC1(CC) 和 CC2(VCONN)
到这里，所有的USB-A to USB-C的线材都可以毙掉了
UART接线定义如下：

UART | Type-C
--|:--
GND | GND
RXD(1.8V) | SBU1
TXD | SBU2
5V | VCC

值得注意的是这里的SBU1和SBU2是区分RX和TX的，所以USB-C不能反插
Pixel 5a 5G和Pixel 2 有不同的USB-C翻转方向
您也可以直接从主板上引出UART，这样就不影响使用ADB了
当然，如果你觉得这样太麻烦，也可以参考[Google开源的工程线](https://github.com/google/usb-cereal)或者[第三方的线](https://github.com/Peter-Easton/android-debug-cable-howto)
注意，以上内容可能不适用于Pixel5(redfin)
