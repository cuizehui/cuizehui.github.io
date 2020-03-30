---
layout:     post
title:      "Android Studio-Flavors"
subtitle:   "使用Flavor添加多分分支渠道打包"
date:       2020-02-28 11:50:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
    - Android-FrameWork
---

# 使用Flavor添加多分支渠道打包

## 增加多版本控制

Flavors 使用，通过版本和渠道号,动态控制哪些代码模块需要带上。达到实现差异化构建

### 步骤1.gradle 中配置

 // 定义两个纬度

    ```gradle
    
     flavorDimensions "apptype", "channel" 
    
     productFlavors{
            develper {
                flavorDimensions "apptype"
                //配置manifest中变量  android:authorities="${TABLE_A_AUTHORITIES}"
                manifestPlaceholders = [TABLE_A_AUTHORITIES: "table_a_authority", TABLE_A_EXPORT: "true"]
            }
            cooperatorA{
                flavorDimensions "apptype"
                manifestPlaceholders = [TABLE_A_AUTHORITIES: "table_a_authority", TABLE_A_EXPORT: "true"]
            }
             Public {
                dimension "channel"
            }
            common {
                dimension "channel"
            }
        } 
    ```    
    
   编译后会生成 BuildConfig 文件
   
    ```java  
    public final class BuildConfig {
      public static final boolean DEBUG = Boolean.parseBoolean("true");
      public static final String APPLICATION_ID = "com.xxxx";
      public static final String BUILD_TYPE = "debug";
      public static final String FLAVOR = "";
      public static final int VERSION_CODE = 201912281;
      public static final String VERSION_NAME = "2.16";
    }
    ```
  
### 步骤2.Java文件中可使用 
    
    ￿``` java   
       public class RcsUtils { 
        public static boolean isCooperatorA() {
            return TextUtils.equals(BuildConfig.FLAVOR_apptype, "flavorDimensions");
        }
       }
    ```  
      
### 依赖库可通过不同版本选择不同库

     ￿``` gradle
    dependencies {
        # ....
        develperCompile "com.android.support:appcompat-v7:25.1.1"
        cooperatorACompile "com.android.support:support-v4:25.1.1"
    }
     ￿```  
### Manifest中配置不同的渠道值 

1.如上Gradle配置
    
    ￿``` gradle
     android:authorities="${TABLE_A_AUTHORITIES}"
     android:exported="${TABLE_A_EXPORT}"     
  ￿  ``` 
  
