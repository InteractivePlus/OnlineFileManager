# OnlineFileManager
A Project for learing upload files using multi-chunks 

# 构想

主框架设计结构： 

伪MVC(React并不具有一般View功能)

即 Model,Views,Controller

Views: React

Controller: NextJS

Model: Golang

预计动工日期：待定

# 多Chunks上传设计

上传协议 gRPC / Websocket

编码格式：Binary

## 处理方案

### 1.根据大小分Chunks并计算Hash，并进行顺序排序

总文件Hash算法：SHA256

Chunk Hash：MurmurHash3

请求接口：

```
/api/RequestUpload
```

参数解释：

```
Token: CSRF Token

Hashes: 文件Hash，用JSON Array编码
排序如下

[ 文件总Hash, Hash1, Hash2, ..., Hashn ]
```

返回值：

```
TaskID: UUID-4格式ID

Status:
成功1，失败0，CSRF错误-1
```

接口作用：

Golang后端会生成一个UUID-4，并在KV数据库新建一个以该UUID为Key的空间，存放该JSON Hash进去

然后JSON Hash解码，每个Hash作为Key新建一个空间

KV数据库可以是Redis/BuntDB


### 2.客户端分Chunks并为每个Chunks计算Hash

首先根据固定大小切分Chunks

切分算法有待考究

随后进行Hash计算

### 3.Async 上传

前端：
新建n个Promise链，然后等待await命令下达

后端：

上传接口：

```
/api/FileUpload
```

参数解释：

```
Token: CSRF Token

UUID: 文件UUID

ChunkHash: 文件Hash

BinaryData: 文件二进制Chunk
```

返回：
```
成功：1
失败：0
Token出错：-1
重传：-2
```

**接口作用：**

在接收到调用后，Golang会通过Channel方式传递给后台Goroutine进行“异步”处理

首先判断ChunkHash是否是文件内部的，通过KV数据库EXISTS方式判断，随后新建一个UUID为名字的文件夹，新建一个ChunkHash为名字的文件

检查到EOF为上传停止，并保存文件

上传过程为实时io.Copy

**判断是否完全上传完毕：**

1.先根据Hash List统计相对应文件数量

2.再统计整个文件夹大小

3.逐个检查Chunk大小和检查ChunkHash

4.按排列顺序进行io.Copy合并

5.检查文件总Hash，删除文件夹



**判断上传失败：**

1.连接突然断开，Golang检查Chunk文件是否成功，失败则删除该Hash文件

2.在KV数据库中相对应的Hash Key标志为Fail或-1

**重传方案：**

在FileUpload请求收到后，Golang会先检查KV中值是否为-1，若为-1则返回-2，要求客户端重传

否则视为成功






