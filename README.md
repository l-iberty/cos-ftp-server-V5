# FTP SERVER 工具

COS FTP SERVER 支持通过FTP协议直接操作COS中的对象和目录，包括上传文件、下载文件、删除文件以及创建文件夹等。

## 开始使用

### 系统环境

操作系统：Linux，推荐使用腾讯CentOS系列CVM

Python解释器版本：Python 2.7

依赖包：

- cos-python-sdk-v5 (>=1.6.5)
- pyftpdlib (>=1.5.2)
- psutil (>=5.6.1)


### 使用限制

适用于COS XML版本


### 安装运行

首先，运行setup.py安装ftp server及其相关的依赖库（需要联网）：

```bash
python setup.py install   # 这里可能需要sudo或者root权限
```

然后，拷贝conf/vsftpd.conf.example 到 conf/vsftpd.conf，参考**配置文件**章节，正确配置bucket和用户信息；

最后，运行ftp_server.py启动cos-ftp-server：

```bash
python ftp_server.py
```

也可以使用nohup命令，以后台进程方式启动：

```bash
nohup python ftp_server.py >> /dev/null 2>&1 &
```

或使用screen命令放入后台运行(需要安装screen工具)：

```bash
screen -dmS ftp
screen -r ftp
python ftp_server.py

Ctrl+A+D 			# 切回主screen即可

```

### 停止

Ctrl + C 即可取消server运行（直接运行，或screen方式放在后台运行）

```bash
ps -ef | grep python | grep ftp_server.py | grep -v grep | awk '{print $2}' | xargs -I{} kill {}
```


## 功能说明

**上传机制**：流式上传，不落本地磁盘，只要按照标准的FTP协议配置工作目录即可，不占用实际的磁盘存储空间。

**下载机制**：直接流式返回给客户端

**目录机制**：bucket作为整个FTP SERVER的根目录，bucket下面可以建立若干个子目录。

**多bucket绑定<sup>*</sup>**：支持同时绑定多个bucket。

**删除操作限制**：在新的ftp-server中可以针对每个ftp用户配置`delete_enable`选项，以标识是否允许该ftp用户删除文件。


*：多bucket绑定是通过不同的ftp server工作路径（`home_dir`）来实现，因此，指定不同的bucket和用户信息时必须保证`home_dir`不同。


### 支持的FTP Server命令

- put
- mput
- get
- rename
- delete
- mkdir
- ls
- cd
- bye
- quite
- size

### 不支持FTP命令

- append
- mget (不支持原生的mget命令，但在某些Windows客户端下，仍然可以批量下载，如FileZilla)

**说明**：Ftp Server工具暂时不支持断点续传功能


## 配置文件

conf/vsftpd.conf.example为Ftp Server工具的配置文件示例，请copy为vsftpd.conf，并按照以下的配置项进行配置：

``` conf
[COS_ACCOUNT_0]
cos_secretid = XXXXXX
cos_secretkey = XXXXXX
cos_bucket = bucketname1-123456789
cos_region = ap-xxx
cos_protocol = https                    # 连接COS或者自定义endpoint所使用的协议类型，默认为https
home_dir = /home/user0
ftp_login_user_name=user0
ftp_login_user_password=pass0
authority=RW
delete_enable=true					# true为允许该ftp用户进行删除操作(默认)，false为禁止该用户进行删除操作
#cos_endpoint = cos.ap-xxx.tencentcos.cn # 设置访问域名，内网访问不需要设置 endpoint，外网访问需要设置成 cos.ap-xxx.tencentcos.cn 外网域名(注意替换xxx为存储桶所在region)

[COS_ACCOUNT_1]
cos_secretid = XXXX
cos_secretkey = XXXXX
cos_bucket = bucketname2-123456789
cos_region = ap-xxx
cos_protocol = https
home_dir = /home/user1
ftp_login_user_name=user1
ftp_login_user_password=pass1
authority=RW
delete_enable=false
#cos_endpoint = cos.ap-xxx.tencentcos.cn

[NETWORK]
masquerade_address = XXX.XXX.XXX.XXX        # 如果FTP SERVER处于某个网关或NAT后，可以通过该配置项将网关的IP地址或域名指定给FTP
listen_port = 2121					   # Ftp Server的监听端口，默认为2121，注意防火墙需要放行该端口

passive_port = 60000,65535             # passive_port可以设置passive模式下，端口的选择范围，默认在(60000, 65535)区间上选择

[FILE_OPTION]
# 默认单文件大小最大支持到200G，不建议设置太大
single_file_max_size = 21474836480

[OPTIONAL]
config_check_enable = true

# 以下设置，如无特殊需要，建议保留default设置  如需设置，请合理填写一个整数
min_part_size       = default
upload_thread_num   = default
max_connection_num  = 512
max_list_file       = 10000                # ls命令最大可列出的文件数目，建议不要设置太大，否则ls命令延时会很高
log_level           = INFO                 # 设置日志输出的级别
log_dir             = log                  # 设置日志的存放目录，默认是在ftp server目录下的log目录中

```

如果要将每个用户绑定到不同的bucket上，则只需要添加`[COS_ACCOUNT_X]`的section即可。

针对每个不同的`COS_ACCOUNT_X`的section有如下说明：

1. 每个ACCOUNT下的用户名（`ftp_login_user_name`）和用户的主目录（`home_dir`）必须各不相同，并且主目录必须是系统中真实存在的目录；
2. 每个COS FTP SERVER允许同时登录的用户数目不能超过100；
3. `endpoint`和`region`不会同时生效，在腾讯云内网使用公有云COS服务只需要正确填写`region`字段即可，`endpoint`常用于外网访问和私有化部署环境中。当同时填写了`region`和`endpoint`，则会`endpoint`会优先生效。

配置文件中的OPTIONAL选项是提供给高级用户用于调整上传性能的可选项，根据机器的性能合理地调整上传分片的大小和并发上传的线程数，可以获得更好的上传速度，一般用户不需要调整，保持默认值即可。

这里提供最大连接数的限制选项：如果不想限制最大连接数，可以填写0，即表示不限制最大连接数目（不过需要根据您机器的性能合理评估）。由于每个ftp客户端和服务器之间会建立控制连接和数据连接，同时ftp server需要保留一个连接用于回复拒绝信息。因此，每个ftp server进程最大能够支持(max_connection_num - 1)/2个客户端同时连接。

在最新版本的cos ftp server程序中提供了配置校验功能，程序会根据当前配置选项评估出运行所需的资源用量，并将其与当前系统可用资源进行对比。若当前系统的可用资源无法满足运行所需，则会拒绝启动cos ftp server进程。该功能可以尽力确保程序在启动后，始终处于资源可控的稳定运行状态。

## FAQ
**填写region后连接不通，怎么配置endpoint**

答：从 2.2.0 版本后，Ftp Server 默认使用内网域名访问 COS，当 Ftp Server 部署在腾讯云 CVM 等内网环境时，只需要设置存储桶所在 region，即可通过内网访问 COS。
当 Ftp Server 部署在外网环境时，需要手动设置`endpoint`字段为`cos.ap-xxx.tecentcos.cn`(其中`ap-xxx`需要替换为具体region)，才能通过外网域名访问 COS

**配置文件中的masquerade_address这个选项有何作用？何时需要配置masquerade_address**

答：当FTP Server运行在一个多网卡的Server，并且FTP Server采用了PASSIVE模式（一般地，FTP客户端位于一个NAT网关之后时，都需要启用PASSIVE模式），此时需要使用masquerade_address选项来唯一绑定一个passive模式下用于reply的IP。
例如，FTP Server有多个IP地址，如内网IP为10.XXX.XXX.XXX，外网IP为123.XXX.XXX.XXX。 客户端通过外网IP连接到FTP Server，同时客户端使用的是PASSIVE模式传输，此时，若FTP Server未指定masquerade_address具体绑定到外网IP地址，则Server在passive模式下，reply时，有可能会走内网地址。就会出现客户端能连接上Ftp server，但是不能从Server端获取任何数据回复。

如果需要配置masquerade_address，建议指定为客户端连接Server所使用的那个IP地址。

**上传大文件的时候，如果中途取消，为什么COS上会留有已上传的文件**

答：由于新版的Ftp Server提供了完全的流式上传特性，如果用户的取消或者断开，都会触发大文件的上传完成操作，因此，COS会认为用户已经数据流上传完成，就会将已经上传的数据组成一个完整的文件。 如果，用户希望重新上传，可以直接以原文件名上传覆盖即可。也可以手动删除不完整的文件，重新上传。

**为什么Ftp Server配置中要设置最大上传文件的限制？**

答：由于COS的分片上传数目最大只能为10000块，且每个分片的大小限制为1MB ~ 5G。 这里设置最大上传文件的限制是为了合理计算一个上传分片的大小。默认支持200G以内的单文件上传，但是不建议用户设置过大，因为单文件大小设置越大，上传时的分片缓冲区也会相应的增大，这可能会耗费用户的内存资源。因此，建议用户根据自己的实际情况，合理设置单文件的大小限制。

**如果上传的文件超过最大限制，会怎么样？**

答：当实际上传的单文件大小超过了配置文件中的限制，则会抛出一个IOError的异常，并且在日志中标注错误信息。
