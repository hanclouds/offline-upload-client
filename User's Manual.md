# 离线上传客户端程序用户手册v1.0.0


离线上传客户端主要通过文件并以CSV（逗号分隔值）格式解析数据并上传至服务端进行处理，目前支持单个文件或者目录的形式上传，平台推荐使用目录多个文件多线程的方式进行数据上传，不推荐使用单个大文件的方式进行数据上传。客户端只会进行基本的CSV数据校验，格式规则如下：

`deviceKey,time,stream,type,data`

在满足上述格式之后，客户端还会对数据流类型与data是否对应做相应的校验，校验不通过则会忽略该条数据，成功则添加至缓冲区等待上传，校验规则如下：

```
type = 0          校验不通过

type = 1          校验data是否为Json

type = 2          校验data是否为整数类型

type = 3          校验data是否为字符串类型（大小不能超过4kb）

type = 4          校验data是否为浮点类型

type = 5          校验data是否为二进制字节流（大小不能超过4kb）

type = 其他值      校验不通过

```

## 开始使用

首先需在平台申请开通离线上传服务，待审核通过后，方可使用该服务。


### 1.下载程序

从[瀚云GitHub](https://raw.githubusercontent.com/hanclouds/offline-upload-client/master/offline-upload-client-1.0.0.tar.gz)下载tar程序包，并解压。程序的目录结构如下：

```
├── offline-upload-client
│   └── bin
│       ├── startup.bat              （Windows启动脚本）
│       ├── startup.sh               （linux启动脚本）
│   └── conf
│       ├── application.properties   （程序配置文件）
│       ├── banner.txt               （banner配置文件）
│       ├── logback.xml              （logback配置文件）
│   └── lib                          （运行类库）
│   └── logs                         （日志目录）
│       ├── app.log
```

### 2.修改配置

应用参数修改

```
vi conf/application.properties
```

```
# 需上传的文件目录路径或文件路径，linux例子：/upload/test  windows例子：D:\\upload\\test
upload.client.path = /upload/test
# 数据所属的产品key
upload.client.product-key =WL4odQnH
# UploadKey
upload.client.upload-key = a0DrJ5GY
# UploadSecret
upload.client.upload-secret =VcpSXwWuj9lfPOLg
# 单次向服务端发送的数量
upload.client.batch-size = 500
# 并行数，单个文件只能由单个线程做处理，建议为目录下文件数量且小等于cpu核心数目的2倍
upload.client.parallel = 3
# 失败重试数，达到最大重试次数后程序将自动退出
upload.client.retrying-time = 3
# 获取token的url链接
upload.client.token-url = http://192.168.1.126:11010/offline/upload/getPermission
# 上传的url链接
upload.client.upload-url = http://192.168.1.126:11010/offline/upload/save
# 上传完成的url链接
upload.client.finish-url = http://192.168.1.126:11010/offline/upload/taskFinish
```

配置参数中必须修改的为`upload.client.path`，`upload.client.product-key`，`upload.client.upload-key`，`upload.client.upload-secret`。其中`upload.client.product-key`必须对应设备所属的productKey，`upload.client.upload-key`及`upload.client.upload-secret`可登陆[平台](https://www.hanclouds.com/)查询，如下图所示：

![upload-client-user](png/upload-client-user.png)

`upload.client.token-url`,`upload.client.upload-url`,`upload.client.finish-url`分别表示服务端暴露的几个接口路径（一般不需要做修改），其他配置参数可根据实际要求参考注释说明自行修改。

### 3.启动程序

本程序依赖JDK1.8，先确保已安装。

linux:

`./bin/startup.sh`

Windows:

`./bin/startup.bat`

程序启动后将会在 `/log`目录下生成日志，默认配置是单个日志最大20MB，保留30天

### 4.断点续传

1.程序在启动后将会向服务端申请临时token，如果获取成功则会记录到`upload.client.path`路径下的`.token`文件中，一个`.token`文件对应的就是一次文件上传任务。如果客户端在完成本地上传任务之前发生中断，就会读取该`.token`文件中的数据进行续传。`.token`文件将会在一次任务完成后自动删除。

2.客户端在获取到token后，会开始数据上传并同样在`upload.client.path`路径下生成`.offset`文件，需要注意的是，每个`.offset`文件都对应一个文件的成功发送数据的行号记录，如果该文件被删除，那么所对应的文件将会从头开始上传。需要注意的是`.offset`文件即使在上传任务完成后也不会自动删除。

3.对于offset的记录，客户端会在每个批次的数据发送成功并获取到服务端的响应后记录到本地的`.offset`文件中，但是如果客户端由于意外的中止（强行关闭），服务端可能会接收到重复数据，最大的重复数据数为：`中断次数 * upload.client.batch-size`

### 5.日志
如果程序出错，可以查阅日志文件：logs/app.log
