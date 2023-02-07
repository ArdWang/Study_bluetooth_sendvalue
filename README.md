# Study_bluetooth_sendvalue
this is Study_bluetooth_sendvalue

小白学习蓝牙二进制的发送与接收数据处理

之前一直没有去写，想着现在来写一下蓝牙数据

Swift版本

形如以下蓝牙文档的要求

```
weight data Attribute Type（UUID）：FFE0。
weight data Fields: 3 bytes 特征值，定义如下：
byte 0，B0：称重重量标志。
B0[7:3]:保留
B0[2]: 称重最大量程。0=5KG；1=10KG；
B0[1:0]: 称重单位。0=KG；1=LB；2和3保留。
byte 1，B1：称重重量低8位。
byte 2，B2：称重重量高8位。

```

1. 由上面可以分析出来这个UUID既可以读取数据，又可以写入数据,总共有3个byte。那么一个byte 有8个比特，即8位组成

所以我们首先看B0,

B0  8位 如下图显示 顺序依次是从右至左读

  ｜    0000_0     ｜     0            ｜  00 
  
  | B0[7:3]保留    ｜ B0[2]称重最大量程  ｜ B0[1:0] 称重单位 

理解这个图就能清除的明白数据怎么接收与发送



2. 来看写入数据，写入数据记得带负号的数据处理

// 0x8000 代表是负号 解析:0x80000首位为1，所以首先肯定是负数

```swift

let value = right >= 0 ? right : 0x8000 + right
let b0 = 0b0000_0000 // 或者 0x00 十六进制的写法
let b1 = value & 0b1111_1111 // 0xff a&b表示a和b按位进行与运算
let b2 = (value & 0b1111_1111_0000_0000) >> 8 // 0xff00
var values = [b0,b1,b2]

var data = Data()
        
for i in 0..<values.count{
    data.append(Data(bytes: &values[i], count: 1))
}
的
// 发送数据
BleManager.shared.writeDValue(data, model?.mReadWeightCharater, model?.mPeripheral)

```
以上代码分析

right 是整数的任何类型 比如: -875

let b0 = 0b0000_0000 主要这么写是由于观察数据方便，一般可以写成0x00代表的是称重单位KG

B0[2]: 称重最大量程。0=5KG；1=10KG；那么最大值的表示如下：
let max = array[0] & 0b0000_0100

协议中 byte 1，B1：称重重量低8位。

低八位的概念：就是16位二进制，也就是：0000 0001 0000 0002。0000 0001 就是高八位，0000 0002就是低八位。

let b1 = value & 0b1111_1111  低八位

& 符号的作用在运行应用程序时，可以看到使用二进制运算符&的输出。对于这个运算符，只有两个输入值都为1时，得到的位才是1 ，否则则为0。

协议中 byte 2, B2: 称重重量高8位

let b2 = (value & 0b1111_1111_0000_0000) >> 8 // 高8位的表示value位运算后需要右移动8位来表示高八位

var values = [b0,b1,b2] 然后把数组拼接起来


3. 接收数据解析

```swift

func dataParser(characteristic: CBCharacteristic) -> String{

    let data = characteristic.value
    let array = data?.toIntArray()

    // 单位
    let unit = array[0] & 0b0000_0011 //0x01

    // 最大值
    let max = array[0] & 0b0000_0100 == 0 ? 5:10 // 0x04

    // 温度值
    let b1 = array[1]
    let b2 = array[2]

    let value = b2 << 8 + b1 // b2 左移 8位做高8位 

    return "0,"+"\(value)"

}


```

kotlin代码：

发送数据 

```kotlin


private fun setWeightData(right:Int){

        val value = if(right >= 0){
            right
        }else{
            0x8000 + right
        }

        val b0 = 0x00
        val b1 = value and 0xff
        val b2 = (value and 0xff00) shr 8

        val ba = byteArrayOf(b0.toByte(),b1.toByte(),b2.toByte())

        BleUtil.get().writeByte(bleModel!!.bleDevice!!,weightService!!,weightCharacter!!,ba)

    }

```

and与&是一摸一样的 shr表示右移动，shl表示左移动

读取数据

```kotlin

fun getNewWeight(bytes: ByteArray): String {

    val unit = bytes[0] and 0x01
    // max重量
    val max = (bytes[0] and (0x04))
    val b1 = bytes[1]
    val b2 = bytes[2]

    val value = b2.toInt() shl 8 or b1.toInt() // 左移动8位

    return "0,$value,$mMax"

}

```
