# GhostTunnel
- 首先这个概念是360提出来的，但是一直没有源码所以我尝试自己复现
- 360的PPT地址（ https://github.com/360PegasusTeam/PegasusTeam/tree/master/talks ）
- 这个项目会正式开源，技术无罪。一直到稳定使用为止。
- 感谢学弟（ https://github.com/MRdoulestar/yunsle_ghost_tunnel ）对我工作前期的启发。
- 非常感谢我的伙伴jerryma0912( https://github.com/jerryma0912 ) 对我开发工作的支持与帮助。不擅长C++的我很大程度受益于他的帮忙。
- 目前windows版本已经进入稳定演示版本。我们测试的平台是windows 10，测试过部分网卡。但是还未记录文档和测试结果。大部分的DOS指令可以成功执行。
- 目前项目已经告一段落


# 说明文档

### 更新记录
**2024年11月15日**
- 注意使用网卡支持的频率，仅支持2.4ghz是不行的
- 现在按照脚本中的代码构造帧结构会导致部分丢失，原因为[微软近期的更新](https://learn.microsoft.com/zh-cn/windows/win32/nativewifi/wi-fi-access-location-changes)
- 在最新微软更新中，还要求给予被控端定位服务，否则会造成API返回ERROR_ACCESS_DENIED
- 代码不再进行维护，有需要自己修改，有其他问题请通过邮件与我联系


### 被控端程序逻辑
被控端程序的目的在于接收控制端的指令，并根据指令执行相应的行为。由于控制端与被控端之间通过Probe request帧和Probe response帧传输，因此被控端逻辑主要分为**【帧传输逻辑】**和**【帧解析逻辑】**两个部分。
#### 帧传输逻辑
Probe request帧由被控端主动发送，并接受控制端发送Probe respense帧。因此，在windows环境下，其执行的逻辑为：获取会话句柄、获取网卡列表及信息、发送request帧、获取附近网络信息列表、获取指定WLAN端口上的网络基本服务集、获取帧数据。

- 获取会话句柄

获取会话句柄采用的api为WlanOpenHandle。该api打开了一个与服务器的连接。只有持有了句柄，才能进行后续才做。

- 获取网卡列表及信息

获取网卡列表采用的api为WlanEnumInterfaces,通过该函数，可以获得一个网卡列表pIfList，通过对该列表的解析，可以获取网卡的信息。

- 发送request帧

在获取网卡信息判断网卡准备无误后，即可通过WlanScan函数发送request帧。发送帧的结构请参见后续部分。

- 获取附近网络信息列表

在发送request帧后，被控端通过WlanGetAvailableNetworkList函数开始扫描附近网络信息，检查是否收到response帧。如果没有，则返回第3步，重新发送request帧，然后再次检测。如果收到response帧,则进入第5步。

- 获取指定WLAN端口上的网络基本服务服务集

利用WlanGetNetworkBssList函数获取指定网络上的BSS列表并进行解析，根据response帧的特征值id过滤出有用的的帧。

- 帧解析

将过滤出的帧并根据帧的定义获取对应的数据，从而完成一次帧传输逻辑。

#### 帧解析逻辑
在完成帧传输逻辑后，即可获取由控制端发送的response帧数据。request帧和response帧的帧结构如下：

##### request帧结构:
```“acc” (3byte) | Hash (8byte)```

名称|作用|
---|:--:|
“acc”|用来标识该帧为请求帧|
Hash |帧第一次发送时间的Hash值|
request帧的前三个字节用于标识该帧为请求帧，后面8个字节为当前帧第一次发送时间的Hash值，为接收端提供帧标识，防止重复接收。

##### response帧结构:

* 执行指令帧： 

```”ccc” (3byte) | Hash (8byte) | Command (244byte)```

名称|作用|
---|:--:|
”ccc”|用来标识该帧为执行指令帧|
Hash |最后一次接收到response帧的Hash值|
Command |执行指令|

执行指令帧的前三个字节用来标识该帧为执行指令帧，后面8个字节为最后一次接收到response帧的Hash值，最后244字节为待执行指令。

* 传输文件帧： 

```”F” (1byte) | FilenameLen (2byte) | Hash (8byte) | FileIndex （2byte） | CurrentIndex (2byte) | FileName+FileContext (220byte)```

名称|作用|
---|:--:|
"F"|用来标识该帧为传输文件帧|
FilenameLen |用来标识文件内容相对于头信息后内容的偏移量|
Hash |当前帧第一次发送时间的Hash值|
FileIndex |文件总分片数|
CurrentIndex |当前接收的分片序号|
FileName+FileContext |先为文件名称，然后为具体的文件内容|

传输文件帧的第1个字节(“F”)用来标识该帧为传输文件帧，第2、3字节(FilenameLen)用来标识文件内容相对于头信息后内容的偏移量，因为文件名的长度是不定的。第4-11字节(Hash)为当前帧第一次发送时间的Hash值，第12、13字节(FileIndex)为文件总分片数，第14、15字节(CurrentIndex)为当前接收的分片序号，第16字节起先为文件名称，然后为具体的文件内容。

根据帧结构，可以很轻易的解析帧所携带的信息。如果帧为执行指令帧，则调用CreateProcess函数创建进程执行指令；如果帧为传输文件帧，则将帧所携带的文件内容写入文件中。

#### 文件说明

* main.cpp /.h                  主程序入口
 
* mainProcess.cpp /.h           主函数逻辑

* Action_ExcuteCmd.cpp /.h      执行命令攻击逻辑

* Action_Sendfile.cpp /.h       发送文件攻击逻辑

### 控制端程序逻辑
response帧由控制端在接收到request帧后发送，因此控制端逻辑主要分为解析命令、抓包、发送帧三个步骤。

- 解析命令

不同的攻击所采用的参数不同。例如:

需要设定自动关机时，所采用的指令是`python main.py -c "shutdown -r -t 60"`

需要远程传输文件时，所采用的指令是`python main.py -f "/root/Desktop/hello.txt"`

因此，需要对攻击者所采用的指令进行解析，从而确定所需的帧结构。

- 抓包

由于reponse帧需要在接收到request帧后对其进行反馈,因此，在python环境下，利用scapy工具包sniff方法对指定网卡进行扫描（注：指定网卡需要切换至monitor模式）。当收到数据包后，根据request帧的特征对数据包的内容进行定位和解析，如果解析到request帧，则将帧的hash部分存入本地缓存。

- 发送帧

根据攻击者的指令，控制端根据定义的response帧结构构造数据包，并按照帧会话逻辑进行发送。


### 帧会话逻辑
由于采用request帧和response帧进行的会话是无连接的，因此很容易造成帧的丢失、重复等问题。因此，在需要多帧传输、单帧验证等使用场景下，需要采用一种可靠的会话逻辑来确保帧的按序传输。帧的会话逻辑如下：

request帧初始状态下Hash值为随机数。在接收到response帧后，将最新接收的response帧的Hash值作为自己的Hash值，从而达到告诉控制端“某帧已经接收到，请发送下一帧”的目的。reponse帧发送的Hash值为第一次该帧发送的时间哈希值，控制端第一次发送500帧，之后每隔4秒钟再增加50帧发送，直到收到request帧携带的Hash与当前帧的Hash值相同为止，否则不发送下一帧。从而达到“确认帧已经送达”的目的。通过这种逻辑，从而保证帧不会丢失。

此外，被控端每次会将最新收到的帧的hash存在本地，每收到response帧时，会将Hash值与本地的hash值进行比较，如果不一致，则执行相关动作，否则，则抛弃该帧。通过这种逻辑，从而保证帧丢失、帧缓存等问题。

考虑到帧所携带的数据量有限，且传输距离较短，因此没有对帧数据进行校验，默认接收到的帧都是正确的，实验测试也表明，没有必要对帧数据进行校验。

### 脚本使用方法
主控脚本采用传统的"参数+指令"方法，其对应参数及含义如下：

```
-h                脚本帮助
-c "command"      远程执行指令，建议执行的指令使用“”括起来，如果没有使用则只能执行单指令
-f "file path"    传输文件,路径可以为绝对路径，也可以为相对路径
```
示例：

```
打开计算器             python main.py "cmd /c calc"
传输hello.txt文件     python main.py "/root/Desktop/hello.txt"
```


### 踩过的坑

1. 程序的整体逻辑比较简单、清晰。帧结构目前仍然有可扩展性，如帧头。

2. 被控端在收到response帧后，会在本地进行缓存，导致被控端反复认为接收到该response帧，因此帧头采用hash值进行标识。

3. 对于被控端，不同的攻击类型所需采用的执行方法不同，如执行命令攻击需要调用createProcess方法来执行命令，而传输文件则需要将帧内容写入文件。因此，对帧头进行了标识，根据标识在客户端采用不同的方法。

4. 控制端的网卡需要切换至monitor模式

5. 在文件多分片传输的情况下，可能会出现network down，为了解决这个问题，后续将开发续传功能。

6. 一次传输完成大约需要3秒，且该时间随着传输距离的增加而增加，因此传输效率低，该问题有待于解决。



