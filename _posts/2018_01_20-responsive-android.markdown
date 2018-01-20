---
layout:     post
title:      "Android 源码编译流程分析"
date:       2018-01-20 14:32:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
    - Android-FrameWork
---

##android 源码编译方式##


###编译环境和配置###
source build/envset.sh
//加载　vendorsetup.sh 脚本
//加载　lunch 的命令（在本次回话窗口有效）
lunch  
// 选择版型　（生成的镜像要运行在什么样的设备上）
//选择好后会存在　answer 中
###整体编译###
make　　－j8  
//整体编译命令执行　　－j8表示线程数（与编译的线程并不是越多越好,通常是根据你机器cup的核心来确定:core*2,即当前cpu的核心的2倍.比如,我现在的笔记本是双核四线程的,因此根据公式,最快速的编译可以make -j8.）



###模块编译指令###
- croot: Changes directory to the top of the tree.
- m: Makes from the top of the tree.
- mm: Builds all of the modules in the current directory.
- mmm: Builds all of the modules in the supplied directories.

针对文件不同查找某个字段的命令　
- cgrep: Greps on all local C/C++ files.
- jgrep: Greps on all local Java files.
- resgrep: Greps on all local res/*.xml files.
- godir: Go to the directory containing a file.

[参考文章](https://www.jianshu.com/p/367f0886e62b)

make snod
重新打包系统镜像
编译好指定模块后,如果我们想要将该模块对应的apk集成到系统镜像中,需要借助make snod指令重新打包系统镜像,这样我们新生成的system.img中就包含了刚才编译的Launcher2模块了.重启模拟器之后生效.





## copy和宏控机制流程 ##
1. 由于原生不支持copy 因此添加脚本将代码进行封装，从而达到以下目的
－　编译流程的简化　只需简单配置文件名
－　客制化　许多重复的修改可以添加宏控达到
－　copy  许多驱动层文件已经配置好直接copy 将原生代码隔离（原型模式）

2. 脚本构建的基本思路
need clone?-> copyfile ->sed　config-> makesagerealAction-> SOAP原生编译方式

3.详细分析

##copy##

[-->file_name: mk]

```
    #!/usr/bin/perl

    #[sagereal][sagereal_COMPILE_ENV]
    $user_mode = "no";
    $clone = "no";
    $action = "new";
    $act = "new";
    $result = 0;

    判断传入参数要执行的动作action和项目名等参数
    .................

    //判断要执行的脚本文件是否存在　　
    //！－e "文件名" 

    if (!-e "../sagereal/script/clone.sh")
    {
      print "../sagereal/script/clone.sh not exst, pls check!\n";
      exit 1;
    }

    if (!-e "../sagereal/mk/$new_project/ProjectConfig.mk")
    {
      print "../sagereal/mk/$new_project/ProjectConfig.mk not exst, pls check!\n";
      exit 1;
    }



    /*open(FILEVAR, "filepath");
            filepath可以有如下三种模式:
                    "filepath"        以只读模式打开文件.
                    ">filepath"       以写模式打开文件.
                    ">>filepath"      以追加模式打开文件,写和追加的区别在于写模式将原文件覆盖,而追加模式则在文件末尾处添加内容.
                    "+>filepath"      以读和写方式打开文件.
                    "+>>filepath"     以读和追加方式打开文件.*/

    /*      当文件打开失败时结束程序 die ("cannot open input file file1\n");*/

    $prj = "../sagereal/mk/$new_project/ProjectConfig.mk";
    open (FILE_HANDLE, "<$prj") or die "cannot open $prj\n";
    while (<FILE_HANDLE>)
    {
      if (/^(\S+)\s*=\s*(\S+)/)
      { 
        //环境变量获取
        $ENV{$1} = $2;
      }
    }
    close FILE_HANDLE;


    ＃如上代码通过只读的方式读取所用宏　并将值存在ENV中下面将其赋值


    my $sagereal_target_project = $ENV{sagereal_target_project};
    my $pcb_config = $ENV{pcb_config};
    my $sagereal_memory_flash  = $ENV{sagereal_memory_flash};
    my $sagereal_audio_effect_para  = $ENV{sagereal_audio_effect_para};
    my $sagereal_camera_effect_para  = $ENV{sagereal_camera_effect_para};
    my $sagereal_battery_capacity  = $ENV{sagereal_battery_capacity};
    my $sagereal_product_folder    = $ENV{sagereal_product_folder};
    #add by Sagereal zhuangshengbin
    my $sagereal_customer  = $ENV{sagereal_customer};
    my $logo_size  = $ENV{logo_size};
    my $wallpaper_size  = $ENV{wallpaper_size};
    my $SAGEREAL_COCLOCK_SUPPORT= $ENV{SAGEREAL_COCLOCK_SUPPORT};


    print "act: $act\n";
    print "user_mode: $user_mode\n";
    print "clone: $clone\n";
    print "sagereal_target_project: $sagereal_target_project\n";
    print "pcb_config: $pcb_config\n";
    print "sagereal_memory_flash: $sagereal_memory_flash\n";
    print "sagereal_audio_effect_para: $sagereal_audio_effect_para\n";
    print "sagereal_camera_effect_para: $sagereal_camera_effect_para\n";
    print "sagereal_battery_capacity: $sagereal_battery_capacity\n";
    print "sagereal_product_folder: $sagereal_product_folder\n";
    print "sagereal_customer: $sagereal_customer\n";
    print "logo_size: $logo_size\n";
    print "wallpaper_size: $wallpaper_size\n";



    //如上代码　将关键信息的配置打印出来　并定义变量　　这些信息　决定了使用那些copy clone 文件的位置，驱动文件的位置，等
    //并并将宏的键和key　存储在环境变量中等待替换


    $file = "../sagereal/mk/$new_project/${sagereal_target_project}_defconfig";
    open (FILE_MODEL, "<$file") or die "cannot open $file\n";
    while (<FILE_MODEL>)
    {
      if (/.*(CONFIG_USB_MTK_OTG)=*\s*(\S+)/)
      {
        $ENV{$1} = $2;
      }
      if (/.*(CONFIG_MTK_CM3623)=*\s*(\S+)/)
      {
        $ENV{$1} = $2;
      }
    }
    close FILE_MODEL;



    ＃此处同理获取项目名　默认值获取


    if ($clone eq "yes")
    {
      print "-------------Copy sagereal files!----------------\n";
        
      //如果需要clone 则执行clone脚本  
      $result = system("bash ../sagereal/script/clone.sh 0 $new_project $pcb_config");
      if ($result != 0)
      {
        print "-------------Copy sagereal files error----------------\n";
        exit 1;
      }
      print "-------------Copy sagereal files end----------------\n";
    }

    if ($act eq "clone")
    {
      print "-----------------------------------------------------\n";
      print "------- sagereal Build Infomation END -------------------\n";
      print "-----------------------------------------------------\n";
    }
    elsif ($act eq "copybin")
    {
      print "-----------------------------------------------------\n";
      print "------- sagereal Build Infomation END -------------------\n";
      print "-----------------------------------------------------\n";

      $result = system("bash ../sagereal/script/build_log.sh $new_project $sagereal_target_project $user_mode");
      $result = system("bash ../sagereal/script/copy_bin.sh $new_project $sagereal_target_project");
    }
    elsif ($act eq "copybin_sign")
    {
      print "-----------------------------------------------------\n";
      print "------- sagereal Build Infomation END -------------------\n";
      print "-----------------------------------------------------\n";

      $result = system("bash ../sagereal/script/build_log.sh $new_project $sagereal_target_project $user_mode");
      $result = system("bash ../sagereal/script/copy_bin_sign.sh $new_project $sagereal_target_project");
    }
    else
    {
    //以上时确定是否clone  /
    //以下执行的是编译脚本　　主要内容在makesagerealAction.sh
      if ($user_mode eq "yes")
      {  
        if($act eq "mm")
        {
           if ( $ARGV[3] eq "" )
           {
            print "mm action need path!\n";
            exit 1;
           }
        }
        print "***************************************************\n";
        print "************** build user software ****************\n";
        print "***************************************************\n";
        print "-----------------------------------------------------\n";
        print "------- sagereal Build Infomation END -------------------\n";
        print "-----------------------------------------------------\n";
        # setBuildEnvVars("./mbldenv.sh");
        $result = system("bash ../sagereal/script/makesagerealAction.sh $user_mode $act $ARGV[3]");
      }
      else
      {
        if($act eq "mm")
        {
          if ( $ARGV[2] eq "" )
          {
           print "mm action need path!\n";
           exit 1;
          }
        }
        print "***************************************************\n";
        print "************** build eng software ****************\n";
        print "***************************************************\n";
        print "-----------------------------------------------------\n";
        print "------- sagereal Build Infomation END -------------------\n";
        print "-----------------------------------------------------\n";
        # setBuildEnvVars("./mbldenv.sh");
        $result = system("bash ../sagereal/script/makesagerealAction.sh $user_mode $act $ARGV[2]");
      }

    //脚本执行完的收尾工作，打印成功或失败异常。

      if ($result == 0)
      {
        print "#### compile exit at: " . &CurrTimeStr . " ####\n";

        if (($act eq "new") 
            || ($act eq "remake") 
            || ($act eq "snod") 
            || ($act eq "pl")
            || ($act eq "rk")
            || ($act eq "update-modem")    
            || ($act eq "newk")
            || ($act eq "lk")
            || ($act eq "logo")
            || ($act eq "kernel")
            || ($act eq "bootimage")
            || ($act eq "otapackage"))
        {
          $result = system("bash ../sagereal/script/build_log.sh $new_project $sagereal_target_project $user_mode");
          $result = system("bash ../sagereal/script/copy_bin.sh $new_project $sagereal_target_project");
        }
        elsif(($act eq "sign-image") || ($act eq "ota-sign"))
        {
          $result = system("bash ../sagereal/script/build_log.sh $new_project $sagereal_target_project $user_mode");
          $result = system("bash ../sagereal/script/copy_bin_sign.sh $new_project $sagereal_target_project");
        }
      }
      else
      {
        print "#### compile exit at: " . &CurrTimeStr . " ####\n";
        print "compile error! Pls check build.log\n";
      }
    }


    sub setBuildEnvVars
    {
      my $bldProfile = shift;
      die "\"$bldProfile\" does NOT exist!\n" if (!-e $bldProfile);
      # Todo: error handling for '. $bldProfile' command
      my $envVarList = `. $bldProfile && env | grep ".*=.*"`;
      map
        {
          chomp;
          $ENV{$1}=$2 if (/(.*)=(.*)/);
        } split(/\n/, $envVarList);

      if ($DEBUG)
      {
        print "[START LOGGING]: Current build environment variables setting...\n";
        print "PATH=$ENV{PATH}\n";
        print "ANDROID_JAVA_HOME=$ENV{ANDROID_JAVA_HOME}\n";
        print "JAVA_HOME=$ENV{JAVA_HOME}\n";
        print "PYTHONPATH=$ENV{PYTHONPATH}\n";
        print "[END LOGGING]\n";
      }
    }

    sub CurrTimeStr
    {
      my($sec, $min, $hour, $mday, $mon, $year) = localtime(time);
      return (sprintf "%4.4d/%2.2d/%2.2d %2.2d:%2.2d:%2.2d", $year+1900, $mon+1, $mday, $hour, $min, $sec);
    }

    #[sagereal][sagereal_COMPILE_ENV]
```



### 分析clone 脚本 ###

[-->file_name:clone.sh]

```
            #!/bin/bash

            ＃modem 处理此处不看
            .................



            ＃软件版本的检查
            echo -e "\e[1;32m Are you sure the SW version is\e[0m\e[1;31m $MTK_BUILD_VERNO\e[0m \e[1;32m? (y or n)\e[0m "
            read -t15 answer
            if [ $answer != "yes" -a $answer != "y" -a $answer != "Y" -a $answer != "Yes" ]; then
            exit 1
            fi

            ＃依次执行clone
            if [ $1 == "0" ]; then
            //拷贝驱动　pcb
              bash ../sagereal/script/clone_drv.sh $2 $3
            fi

            //拷贝客制化
            bash ../sagereal/script/clone_integration.sh $2
            //copy mmi
            bash ../sagereal/script/clone_mmi.sh $2
```



### 分析copy mmi ###
此处按照猜测应该分为两部分　，一部分copy替换文件，一部分替换宏控

[-->file_name:clone_mmi.sh]
``` 
            //此处为宏控字段替换的核心函数
            function write_xml()
            {
            change=${!1}
            if [ $change ]; then
               if [ $change = "sr_null" ];then
            #     sed -i "s/\"$control\">.*</\"$control\"></" $path
                  sed -i "/\"$1\"/s/\">.*</\"></" $path
               else
            #     sed -i "s/\"$control\">.*</\"$control\">$change</" $path
                  sed -i "/\"$1\"/s/\">.*</\">$change</" $path
               fi
               echo "$1 = $change"
            #else
            #  echo "$control is not be setted"
            fi
            }
```



















