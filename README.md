# dell-inspiron7590-hackintosh
灵越7590的黑苹果配置
## 配置
灵越7590顶配, 4K版本。

CPU: i7 9750H

RAM: 32GB DDR4

SSD1: 原厂东芝BG4 512GB (Windows)

SSD2: SN500 nvme (macOS)

Graphic card: Intel UHD 630， NVIDIA Geforce GTX 1650（已屏蔽)

WIFI: DW1560 已驱动

Bluetooth: 已驱动

## 正常的功能

* CPU 正常变频

* hiDPI正常

* 正常睡眠, 正常关机重启 (已经禁用休眠）

* Fn键功能键(声音，键盘背光调节，亮度调节)正常

* 触控板正常，模式是GPIO中断,手势全部支持。插电的时候比较流畅，电池模式下会有漂移现象，比较诡异。

* 声卡正常驱动

* 雷点3接口仅支持使用Typc-C设备

## 无法驱动的功能

* 独立显卡，无解

* HDMI和雷电3的dp alternate mode，都无法使用，因为是独显输出信号的

* FaceTime，handoff，imessage无法使用，后期再去调整

## 有问题的功能

* 把VirtualSMC替换成FakeSMC后睡眠重启的几率大大降低，但是仍然存在睡眠无法唤醒的问题。

## DSDT的补丁

#### 触控板
* 这台电脑的IOnterruptSpecifier数值是0x33，但是在macOS下无法查看，所以很多同学以为打上voodool就可以完美驱动，其实并不是的，这样直接打上去还是轮询模式。
直接怼上我改GPIO的代码，把这里整个替换TPD1,如果你的IOnterruptSpecifier也是0x33，我觉得是可以用的。

``` 

Device (TPD1)
        {
            Name (HID2, Zero)
            Name (SBFB, ResourceTemplate ()
            {
                I2cSerialBusV2 (0x002C, ControllerInitiated, 0x00061A80,
                    AddressingMode7Bit, "\\_SB.PCI0.I2C1",
                    0x00, ResourceConsumer, _Y3F, Exclusive,
                    )
            })
            Name (SBFG, ResourceTemplate ()
            {
                GpioInt (Level, ActiveLow, ExclusiveAndWake, PullDefault, 0x0000,
                    "\\_SB.PCI0.GPI0", 0x00, ResourceConsumer, ,
                    )
                    {   // Pin list
                        0x0023
                    }
            })
            CreateWordField (SBFB, \_SB.PCI0.I2C1.TPD1._Y3F._ADR, BADR)  // _ADR: Address
            CreateDWordField (SBFB, \_SB.PCI0.I2C1.TPD1._Y3F._SPE, SPED)  // _SPE: Speed
            CreateWordField (SBFG, 0x17, INT1)
            Method (_INI, 0, NotSerialized)  // _INI: Initialize
            {
                If (LLess (OSYS, 0x07DC))
                {
                    SRXO (GPDI, One)
                }

                Store (GNUM (GPDI), INT1)
                If (LEqual (SDM1, Zero))
                {
                    SHPO (GPDI, One)
                }

                Store (0x20, HID2)
                Return (Zero)
            }

            Method (_HID, 0, NotSerialized)  // _HID: Hardware ID
            {
                If (LEqual (CBSN, One))
                {
                    Return ("DELL0922")
                }
                ElseIf (LEqual (CBSN, 0x02))
                {
                    Return ("DELL0923")
                }
                ElseIf (LEqual (CBSN, 0x03))
                {
                    Return ("DELL0924")
                }
                Else
                {
                    Return ("DELL0922")
                }
            }

            Name (_CID, "PNP0C50")  // _CID: Compatible ID
            Name (_S0W, 0x03)  // _S0W: S0 Device Wake State
            Method (_DSM, 4, Serialized)  // _DSM: Device-Specific Method
            {
                If (LEqual (Arg0, HIDG))
                {
                    Return (HIDD (Arg0, Arg1, Arg2, Arg3, HID2))
                }

                If (LEqual (Arg0, TP7G))
                {
                    Return (TP7D (Arg0, Arg1, Arg2, Arg3, SBFB, SBFG))
                }

                Return (Buffer (One)
                {
                     0x00                                          
                })
            }

            Method (_STA, 0, NotSerialized)  // _STA: Status
            {
                Return (0x0F)
            }

            Method (_CRS, 0, NotSerialized)  // _CRS: Current Resource Settings
            {
                Return (ConcatenateResTemplate (SBFB, SBFG))
            }
        }

``` 

### 屏幕背光亮度调节的键盘映射
* 这部分参考了大神的帖子https://www.tonymacx86.com/threads/guide-patching-dsdt-ssdt-for-laptop-backlight-control.152659/
* 这台机器背光调节是F6, F7。通过监控console发现这两个键点击走的是ACPI而不是PS2控制器，走的是APCI的BRT6方法。为了能让点击事件被PS2识别，我们需要改BRT6方法让它能够映射到PS2的brightness up down，这两个键分别是0x0405(F14)，0x0406(F15).
进DSDT，找到BRT6方法。

``` 
//修改的
Method (BRT6, 2, NotSerialized)
        {
            If (LEqual (Arg0, One))
            {
                //F6点击时通知0x0405键,(即F14)，F14被PS2接管了
                Notify (^^LPCB.PS2K, 0x0405)
            }

            If (And (Arg0, 0x02))
            {
                //F7点击时通知0x0406键,(即F15)，F15被PS2接管了
                Notify (^^LPCB.PS2K, 0x0406)
            }
                
        }
   
//原来的
Method (BRT6, 2, NotSerialized)
        {
            If (LEqual (Arg0, One))
            {
                Notify (LCD, 0x86)
            }

            If (And (Arg0, 0x02))
            {
                Notify (LCD, 0x87)
            }
        }
        
        
``` 

## 一些说明
* BrcmBluetoothInjector.kext 如果系统版本不是10.14.6或者以上，请屏蔽或者删除。
* 除了FakeSMC.kext放入/kext/others，其它驱动应该放入/Library/Extensions，然后重建缓存。
        
        
