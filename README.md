
# Tuya Mesh 通用串口接入协议
**文档介绍：** 参考tuya mesh通用串口对接协议。由于mesh协议中很多走的是特殊的命令，因此单独拉出来做一个串口通用接入协议，满足想要自己实现灯算法控制的客户。

**特别说明：**

**参考文档：**

《涂鸦wifi低功耗通用串口接入协议》
《通用固件协议修改历史说明.docx》
《Tuya Mesh 低功耗通用串口接入协议（1.0.1）.md》

**阅读工具：**

http://www.beautifulzzzz.com/mdedit/

**版本说明：**

版本号|描述|作者|时间|审核人|状态
---|---|---|---|---|---
1.0.0|初始创建|李涛|2018-04-18 | |
1.0.1|不采用DP方式上报/下发数据，而是采用mesh通信格式|李涛|2018-04-26 | |
1.0.2|修改3.2、获取产品信息，支持MCU上报VendorId；添加第五章，灯上报举例 | litao | 2018-05-03 | |
1.0.3|添加3.10数据透传上报（广播）命令，方便用户实现各种自定义联动逻辑 | litao | 2018-05-22 | |
1.0.4|添加增加/删除当前模块组地址协议 | litao | 2018-08-16 | |
1.0.5|添加当前模块地址查询协议 | 章敏煜 | 2018-08-20| |
1.0.6|添加模块产测功能 | 章敏煜 | 2018-11.19| |

![][#bar]
### 1 串口通信约定

波特率：9600
数据位：8
奇偶校验：无
停止位：1
数据流控：无
MCU：用户控制板控制芯片，与涂鸦模块通过串口对接

![][#bar]
### 2 格式说明

字段 | 长度（byte）| 说明 
--- | --- | --- 
帧头 | 2 | 固定为0x55aa
版本 | 1 | 升级扩展用
命令字 | 1 | 具体帧类型
数据长度 | 2 | 大端
数据 | N | 
校验和 | 1 | 从帧头开始按字节求和得出的结果对256求余

说明：

- 所有大于1个字节的数据均采用大端模式传输。
- 一般情况下，采用同命令字一发一收同步机制，即一方发出命令，另一方应答，若发送方超时未收到正确的响应包，则超时传输，如下图所示：
      
![][#1]


说明：具体通信方式以“协议详述”章节中为准

- MCU状态上报则采用同步模式，MCU状态上报“命令字”为y，如下所示：(MCU统计数据上报)

![][#2]
       
说明：下面是整个通信流程

- 命令下发流程只有： - 3.6 - APP->MESH->串口直接给MCU（命令下发透传）
- 状态上报有以下几种情况：
	- 统计数据上报 - 3.5 -（该命令上报的数据，会送到mesh芯片的缓存中，当手机有GET请求时，会将对应信息返回给手机）
	- 主动通知上报 - 3.7 -（特殊的，只有1BYTE可以使用，用来向所有mesh节点时时同步重要信息，灯的第1byte表示亮度，亮度为0表示关灯；插座也有特殊要求，见下面的说明）
	- 命令上报透传 - 3.10 -（该命令会直接转换为mesh广播命令，发送到mesh网络）
	

![][#3]

![][#bar]
### 3 协议详述
#### 3.1、心跳检测
说明：

- 1) 模块上电后,以 10s 的间隔定期发送心跳,若在超时时间( 3s )内,未收到 MCU 的回应,则认为 MCU 离线。
- 2) MCU 侧也可依据此心跳定期检测模块是否正常工作,若模块无心跳下发,则 MCU 可认为模块异常。
- 3) 模块通过心跳检测到 MCU 重启需要重新读取 mesh 网络所有节点在线状态。

模块发送:

字段 | 长度（byte）| 说明 
--- | --- | --
帧头 | 2 | 0x55aa
版本 | 1 | 0x00
命令字 | 1 | 0x00
数据长度 | 2 | 0x0000
数据 | 0 | 无
校验和 | 1 | 从帧头开始按字节求和得出的结果对256求余

MCU 返回:

字段 | 长度（byte）| 说明 
--- | --- | ---
帧头 | 2 | 0x55aa
版本 | 1 | 0x01
命令字 | 1 | 0x00
数据长度 | 2 | 0x0001
数据 | xxx | 0x00 : MCU 重启后第一次心跳返回值,仅发送
| | 一次,用于模块判断工作过程中MCU是否重启
| | 0x01 :除 MCU 重启后第一次返回0外,其余均
| | 返回此值
校验和 | 1 | 从帧头开始按字节求和得出的结果对256求余

#### 3.2、获取产品信息
说明：

- 1) 产品信息由product key、MCU软件版本、vendorId构成
- 2) product key：对应涂鸦开发者平台PID(产品标识)，由涂鸦云开发者平台生成，用于云端记录产品相关信息
- 3) MCU软件版本号格式定义：采用点分十进制形式，”x.x.x”（0<=x<=99），x为十进制数
- 4) VendorId是产品类别标志：类别分为大类和小类，大类有：灯大类（0x01）、电工大类（0x02）、传感器大类（0x04）、适配器大类（0x08）、执行器大类（0x10）；每种大类下又有多种小类：灯大类下有1～5路灯（0x01～05）、插座下有1～6路排插（0x01～06）、传感器类下有（门磁、PIR、亮度等）... 例如RGBL四路灯的vendorID为：0104

模块发送：

字段 | 长度（byte）| 说明
--- | --- | ---
帧头 | 2 | 0x55aa
版本 | 1 | 0x00<模块版本字永远为0x00>
命令字 | 1 | 0x01
数据长度 | 2 | 0x0000
数据 | 0 | 无
校验和 | 1 | 从帧头开始按字节求和得出的结果对256求余

MCU返回：

字段 | 长度（byte）| 说明
--- | --- | ---
帧头 | 2 | 0x55aa
版本 | 1 | 0x01<MCU存在多个版本字>
命令字 | 1 |0x01
数据长度 | 2 | N
数据 |  N | {“p”:”abcdefgh”,“v”:”1.0.0”}
校验和 | 1 | 从帧头开始按字节求和得出的结果对256求余

例：{“p”:”abcdefgh”,“v”:”1.0.0”,"k":"0104"}
p 表示产品ID为abcdefgh
v 表示MCU版本为1.0.0
k 表示产品类别为4路灯

#### 3.3、报告设备联网状态 *

设备联网状态 | wifi-描述 |状态值|mesh-描述|状态值
--- | --- | --- | --- | ---
状态1 | smartconfig配置状态 | 0x00 | OUT-MESH | 0x00
状态2 | AP配置状态 | 0x01 | |
状态3 | WIFI已配置但未连上路由器 | 0x02 | |
状态4 | WIFI已配置且连上路由器 | 0x03 | |
状态5 | 已连上路由器且连接到云端 | 0x04 | IN-MESH | 0x04
状态6 |  |  | factory test | 0x05

说明：

- 1）设备联网状态：OUT-MESH 处于未配网状态、IN-MESH 处于已配网状态。未配网时工作模式指示灯快闪间隔250ms，已配网模式灯常亮。为了和wifi兼容，mesh不用2,3,4状态。
- 2）当模块的状态发生变化，则主动下发mesh状态至MCU。

字段 | 长度（byte）| 说明 
--- | --- | --
帧头 | 2 | 0x55aa
版本 | 1 | 0x00
命令字 | 1 |0x02
数据长度 | 2 | 0x0001
数据 | 1 | 指示WIFI工作状态：
| |0x00:状态1
| |0x01:状态2
| |0x02:状态3
| |0x03:状态4
| |0x04:状态5
校验和 | 1 | 从帧头开始按字节求和得出的结果对256求余

MCU返回：

字段 | 长度（byte）| 说明 
--- | --- | --
帧头 | 2 | 0x55aa
版本 | 1 | 0x01
命令字 | 1 | 0x02
数据长度 | 2 | 0x0000
数据 | 0 | 无
校验和 | 1 | 从帧头开始按字节求和得出的结果对256求余

#### 3.4、重置设备 *
MCU向模块发送重置命令，模块重置到出厂默认设置。wifi有重置选择配置模式，mesh里没有。

MCU发送：

字段 | 长度（byte）| 说明 
--- | --- | --
帧头 | 2 | 0x55aa
版本 | 1 | 0x01
命令字 | 1 | 0x03
数据长度 | 2 | 0x0000
数据 | 0 | 无
校验和 | 1 | 从帧头开始按字节求和得出的结果对256求余

模块返回：

字段 | 长度（byte）| 说明 
--- | --- | --
帧头 | 2 | 0x55aa
版本 | 1 | 0x00
命令字 | 1 | 0x03
数据长度 | 2 | 0x0000
数据 | 0 | 无
校验和 | 1 | 从帧头开始按字节求和得出的结果对256求余

#### 3.5、MESH状态上报（到MESH模块缓存）（upload)

说明：

- 当灯、插座、传感器等功能点的数据发生变化时，需要合成相应的数据包将信息同步到MESH模块

特别的，对于灯有下列要求：

- 1)当设备的开关/亮度/R/G/B/W/C/Mode/Scence1～8发生变化时要上报


MCU发送：

字段 | 长度（byte）| 说明 
--- | --- | --
帧头 | 2 | 0x55aa
版本 | 1 | 0x01
命令字 | 1 | 0x05
数据长度 | 2 | N
数据 | N | 
校验和 | 1 | 从帧头开始按字节求和得出的结果对256求余

模块返回：

字段 | 长度（byte）| 说明 
--- | --- | --
帧头 | 2 | 0x55aa
版本 | 1 | 0x00
命令字 | 1 | 0x05
数据长度 | 2 | 0x00
数据 | 0 | NULL
校验和 | 1 | 从帧头开始按字节求和得出的结果对256求余

#### 3.6、MESH命令透传下发

说明：

- 当有控制命令从手机或者网关发送到mesh节点模块时，模块通过串口将命令发送给MCU

模块返回：

字段 | 长度（byte）| 说明 
--- | --- | --
帧头 | 2 | 0x55aa
版本 | 1 | 0x00
命令字 | 1 | 0x06
数据长度 | 2 | N
数据 | N | 未经过转换的mesh原始数据
校验和 | 1 | 从帧头开始按字节求和得出的结果对256求余

MCU发送：

字段 | 长度（byte）| 说明 
--- | --- | --
帧头 | 2 | 0x55aa
版本 | 1 | 0x01
命令字 | 1 | 0x06
数据长度 | 2 | 0x0000
数据 | 0 | NULL
校验和 | 1 | 从帧头开始按字节求和得出的结果对256求余



#### 3.7、MESH通知信息变化上报（notify）

说明：
- 3.5同步的信息是方便手机端请求设备状态用的，这条协议上报的数据是用来主动推送到手机的，且由于MESH本身限制，该信息只有1字节

特别的：

- 灯：1Byte  表示灯的当前亮度值，如果灯关闭，该值为0
- 插座：1Byte 表示插座当前每个子插的状态，最左边的第一bit的0/1表示第一路子插的关开状态，依次类推

MCU发送：

字段 | 长度（byte）| 说明 
--- | --- | --
帧头 | 2 | 0x55aa
版本 | 1 | 0x01
命令字 | 1 | 0x07
数据长度 | 2 | 0x0001
数据 | 1 | 1Byte
校验和 | 1 | 从帧头开始按字节求和得出的结果对256求余

模块返回：

字段 | 长度（byte）| 说明 
--- | --- | --
帧头 | 2 | 0x55aa
版本 | 1 | 0x00
命令字 | 1 | 0x07
数据长度 | 2 | 0x00
数据 | 0 | NULL
校验和 | 1 | 从帧头开始按字节求和得出的结果对256求余


#### 3.8、MESH 所有状态请求

说明：

- 当模块

模块返回：

字段 | 长度（byte）| 说明 
--- | --- | --
帧头 | 2 | 0x55aa
版本 | 1 | 0x00
命令字 | 1 | 0x08
数据长度 | 2 | 0x0000
数据 | 0 | NULL
校验和 | 1 | 从帧头开始按字节求和得出的结果对256求余

MCU发送：按照上报命令UPLOAD+NOTIFY，上报所有状态数据


#### 3.9、MESH功能性测试

说明：扫描指定mesh广播节点的SSID: ty_mdev,返回扫描结果和信号强度百分比

MCU发送：

字段 | 长度（byte）| 说明 
--- | --- | --
帧头 | 2 | 0x55aa
版本 | 1 | 0x01
命令字 | 1 | 0x09
数据长度 | 2 | 0x0000
数据 | Data | 无
校验和 | 1 | 从帧头开始按字节求和得出的结果对256求余

模块返回：

字段 | 长度（byte）| 说明 
--- | --- | --
帧头 | 2 | 0x55aa
版本 | 1 | 0x00
命令字 | 1 | 0x09
数据长度 | 2 | 0x0002
数据 | Data | 数据长度为2字节：
| | Data[0]:0x00失败, 0x01成功;
| | 当Data[0]为0x01，即成功时，Data[1]表示信号强度 (0-100, 0信号最差，100信号最强)
| | 当Data[0]为0x00，即失败时，Data[1]为0x00表示未扫描到指定的ssid，Data[1]为0x01表示模块未烧录授权key
校验和 | 1 | 从帧头开始按字节求和得出的结果对256求余


#### 3.10、MESH状态透传上报（Ad 广播）

说明：

- 该命令一般情况下不建议使用，除非比较了解整个tuya mesh协议
- 该命令可以实现MCU合成原始mesh控制命令数据包，经过串口通过mesh模块，直接透传到mesh网络，直接对mesh网络设备进行控制
- 该命令一般用于客户想要实现简单的联动逻辑：比如 —— 墙面开关和灯的联动，墙面开关直接广播灯开关控制命令

MCU发送：

字段 | 长度（byte）| 说明 
--- | --- | --
帧头 | 2 | 0x55aa
版本 | 1 | 0x01
命令字 | 1 | 0x04
数据长度 | 2 | N
数据 | N | 
校验和 | 1 | 从帧头开始按字节求和得出的结果对256求余

模块返回：

字段 | 长度（byte）| 说明 
--- | --- | --
帧头 | 2 | 0x55aa
版本 | 1 | 0x00
命令字 | 1 | 0x04
数据长度 | 2 | 0x00
数据 | 0 | NULL
校验和 | 1 | 从帧头开始按字节求和得出的结果对256求余

注：具体的数据格式见附录4
注：透传上报，连续两包之间的时间间隔要大于350ms，切忌暴力广播


#### 3.11、串口操作当前设备的地址协议（高级操作）
#### 3.11.1、组地址增加删除

说明：

- 该命令用于给当前设备增加、删除组地址用
- mesh组地址的规则为：单个节点最多属于8个组，总共可选组地址为0x8000~0xFFFE

MCU发送：

字段 | 长度（byte）| 说明 
--- | --- | --
帧头 | 2 | 0x55aa
版本 | 1 | 0x01
命令字 | 1 | 0xB1
数据长度 | 2 | 0x00 0x03
数据 | 3 |  0x01（增加）/ 0x00（删除）+ 2字节组地址（0x8001则表示为：0x80 0x01)
校验和 | 1 | 从帧头开始按字节求和得出的结果对256求余

模块返回：

字段 | 长度（byte）| 说明 
--- | --- | --
帧头 | 2 | 0x55aa
版本 | 1 | 0x00
命令字 | 1 | 0xB1
数据长度 | 2 | 0x00 0x01
数据 | 1 | 0x00（失败） 0x01（成功）0x03 (当前设备群组地址已满)
校验和 | 1 | 从帧头开始按字节求和得出的结果对256求余

#### 3.11.2、组地址查询

说明：

- 该命令用于查询当前设备所处于的群组

MCU发送：

字段 | 长度（byte）| 说明 
--- | --- | --
帧头 | 2 | 0x55aa
版本 | 1 | 0x01
命令字 | 1 | 0xB1
数据长度 | 2 | 0x00 0x01
数据 | 3 |  0x02（查询）
校验和 | 1 | 从帧头开始按字节求和得出的结果对256求余

模块返回：

字段 | 长度（byte）| 说明 
--- | --- | --
帧头 | 2 | 0x55aa
版本 | 1 | 0x00
命令字 | 1 | 0xB1
数据长度 | 2 | 0x00 0x11
数据 | 1 |  0x02（查询）8个2字节当前设备群组地址
校验和 | 1 | 从帧头开始按字节求和得出的结果对256求余

注：设备别当前群组地址在移除设备或者重置设备后都将别备清除
![][#bar]
### 4 附录
说明：

- MESH命令透传下发数据格式：

![][#4]

- MESH透传上报（Ad广播）数据的格式：

NO. | 1 | 2 | 3 | 4| 5 | 6 | 7 | 8 | 9 ~ 18 
--- | --- | --- | --- |  --- | --- | --- | --- | --- | ---
data | Sn0 | Sn1 | Sn2 | dst1 | dst2 | cmd | uuid(大类） | uuid（小类） | params(10)

注：dst为命令广播给mesh节点的地址（目的地址)，为16字节数据：如0x0807,采用大端格式则dst1 = 07,dst2 = 08

- MESH状态上报（到MESH模块缓存）（upload)数据的格式：

NO. | 1 | 2 | 3 | 4 ~ 13 
--- | --- | --- | --- |  ---
data | uuid（大类） | uuid（小类） | cmd | params(10)

- MESH通知信息变化上报（notify）数据的格式：

说明：notify上报只有1BYTE，对于灯是其亮度数值（0～100，关灯时值为0），对于插座1字节的8bits从左到右的0/1值表示子插1～6的关开


### 5 上报举例
#### 5.1、灯大类
上报类型 |帧头 | 版本 | 命令字 | 数据长度 | 数据 | 校验和 | 意义 | 纯属据 
--- | --- | --- | --- | --- | --- | --- | --- | ---
一路灯状态upload |55 AA  | 01 | 05 | 00 0D | 01 01 DB 00 00 00 00 00 32 00 04 00 00 |  25 | 上报1路灯亮度为50%（R=G=B=W=C=S=0,V=04表明只有亮度值有效，最后面两个0补够10params) | 55 AA 01 05 00 0D 01 01 DB 00 00 00 00 00 32 00 04 00 00 25 
三路灯状态upload | 55 AA | 01 | 05 | 00 0D | 01 03 DB FF 00 00 00 00 64 00 E0 00 00 | 34 | 上报3路灯红色 | 55 AA 01 05 00 0D 01 03 DB FF 00 00 00 00 64 00 E0 00 00 34
灯notify | 55 AA | 01 | 07 | 00 01 | 64 | 6C | notify亮度为100% | 55 AA 01 07 00 01 64 6C
灯notify | 55 AA | 01 | 07 | 00 01 | 32 | 3A | notify亮度为50% | 55 AA 01 07 00 01 32 3A 
 四路灯请求设备信息答复 | 55 AA | 01 | 01 | 00 27 | 7B 22 70 22 3A 22 48 75 6D 39 48 55 46 74 22 2C 22 76 22 3A 22 31 2E 30 2E 30 22 2C 22 6B 22 3A 22 30 31 30 34 22 | 7D | "\{"p":"Hum9HUFt","v":"1.0.0","k":"0104"\}" | 55 AA 01 01 00 27 7B 22 70 22 3A 22 48 75 6D 39 48 55 46 74 22 2C 22 76 22 3A 22 31 2E 30 2E 30 22 2C 22 6B 22 3A 22 30 31 30 34 22 7D
三路灯请求设备信息答复 | 55 AA | 01 |  01 |  00 27 | 7B 22 70 22 3A 22 34 31 79 71 56 44 59 35 22 2C 22 76 22 3A 22 31 2E 30 2E 30 22 2C 22 6B 22 3A 22 30 31 30 33 22 7D | 37 | "\{"p":"41yqVDY5","v":"1.0.0","k":"0103"\}"  |55 AA 01 01 00 27 7B 22 70 22 3A 22 34 31 79 71 56 44 59 35 22 2C 22 76 22 3A 22 31 2E 30 2E 30 22 2C 22 6B 22 3A 22 30 31 30 33 22 7D 37
reset复位重置命令| 55 AA | 01 | 03 | 00  00 | |  03 | 重置设备至可配网状态 |55 AA 01 03 00 00 03

----
- 三路灯基本操作：

步骤 | 码
--- | ---
1）重置：| 55 AA 01 03 00 00 03
2）发送第一包心跳响应：| 55 AA 01 00 00 01 00 01
3）上报设备信息： | 55 AA 01 01 00 27 7B 22 70 22 3A 22 34 31 79 71 56 44 59 35 22 2C 22 76 22 3A 22 31 2E 30 2E 30 22 2C 22 6B 22 3A 22 30 31 30 33 22 7D 37
4）上报notify：| 55 AA 01 07 00 01 64 6C
5）上报RGBWCL：| 55 AA 01 05 00 0D 01 03 DB FF 00 00 00 00 64 00 E0 00 00 34

注：之后每次开关变化（开：亮度为100；关：亮度为0）、RGB变化，都要上报notify和upload

- 两路灯基本操作：

步骤 | 码
--- | ---
1）重置：| 55 AA 01 03 00 00 03
2）发送第一包心跳响应：| 55 AA 01 00 00 01 00 01
3）上报设备信息： | 55 AA 01 01 00 27 7B 22 70 22 3A 22 74 4C 33 4D 50 75 6B 37 22 2C 22 76 22 3A 22 31 2E 30 2E 30 22 2C 22 6B 22 3A 22 30 31 30 32 22 7D 66
4）上报notify：| 55 AA 01 07 00 01 32 3A
5）上报RGBWCL：| 55 AA 01 05 00 0D 01 02 DB 00 00 00 7F 80 32 00 1C 00 00 3D


[#bar]:https://btfzmd.oss-cn-shanghai.aliyuncs.com/base/bar.png
[#1]:http://tuchuang.beautifulzzzz.com:3000/?path=20180418/1.png
[#2]:http://tuchuang.beautifulzzzz.com:3000/?path=20180418/2.png
[#3]:http://tuchuang.beautifulzzzz.com:3000/?path=20180522/2.png
[#4]:http://tuchuang.beautifulzzzz.com:3000/?path=20180428/1.png


