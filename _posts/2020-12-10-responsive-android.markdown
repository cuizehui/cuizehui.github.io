---
layout:     post
title:      "Android逆向工程-备份技术"
subtitle:   "非Root手机获取沙盒包内容"
date:       2020-10-11 11:58:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
- Android-Application
---
# Android备份技术

## 备份工具 adb 
    
    adb backup [-f <file>] [<packages...>
   
    adb backup -f /Users/nela.cui/Desktop/123.ab com.android.messaging 
  
## 解包工具 android-backup-extractor
    
    unpack:	abe unpack	<backup.ab> <backup.tar> [password]
    pack:		abe pack	<backup.tar> <backup.ab> [password]
    pack for 4.4:	abe pack-kk	<backup.tar> <backup.ab> [password]
    
    java -jar abe.jar unpack /Users/nela.cui/Documents/safe/Backup/backup.ab backup.tar 

## 解包后文件内容

- _manifest	配置文件
- ef 外部存储文件
- r
- db 数据库文件
- f file文件夹
- sp sharePreference

## 修改解包后的文件还原备份

### DD命令 去掉ab头

命令用于读取、转换并输出数据。

if=文件名：输入文件名，默认为标准输入。即指定源文件。
of=文件名：输出文件名，默认为标准输出。即指定目的文件。
bs=bytes：同时设置读入/输出的块大小为bytes个字节。

dd if=backup.ab路径 bs=24 skip=1| openssl zlib -d > backup.tar

若无 zlib 模块可使用python 

dd if=backup.ab路径 bs=1 skip=24 | python -c "import zlib,sys;sys.stdout.write(zlib.decompress(sys.stdin.read()))" > backup.tar

### 记录文件顺序还原ab文件

tar -tf backup.tar > backup.list 


## warning 

WARNING: adb backup is deprecated and may be removed in a future release
Now unlock your device and confirm the backup operation..

此功能与Manifest 属性有关

android:allowBackup="true"

## 操作脚本

```
!/bin/bash


 name=$1

 fileName=${name//\./_}
 if [ ! -f "output/$fileName.ab" ];then
 	echo "请点击屏幕获取备份文件"
 	adb backup -f  output/$fileName.ab $1
 else
 	echo "正在解压备份文件"
  	java -jar abe.jar unpack output/$fileName.ab output/$fileName.tar 
  	# tar -xf output/$fileName.tar
 fi
```