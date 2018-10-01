---
layout:     post
title:      "接口自动化测试(一)--反射模块"
date:       2018-09-25 12:56:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
- AndroidSDK-Test
---

# 接口自动化测试(一)--反射模块

## 整体流程

由于依赖sdk的模块间解耦和接口化

1. 测试脚本获取测试参数。xml解析。 服务器发送测试数据。
2. 初始化要测试的模块，初始化步骤由于参数众多所以不采用反射方式。
3. 通过socket网络模块 拿到测试数据（json数据）。选定要测试的类。
4. 解析测试数据，确定要测试的模块、方法名、参数列表
5. 通过反射调用。将调用结果通过网络模块回传。
6. 通过动态代理，代理回调，封装回调结果，通过网络模块发送至服务器。
 
## 模块

最终将程序拆分为4大模块

- json数据解析模块 
- socket数据接收发送模块
- 反射调用模块
- 回执数据组json
- 代理回调接口模块

------------

## 解析模块

### 基本需求
根据给定数据结构json,构造的实体类对象。提供给发射模块调用。
例如定义的测试数据如下结构：

```
{
    "type":"command",
    "module":"client",
    "method":"login",
    "params":[
        {
            "string":"423424"
        },
        {
            "string":"234234354"
        }
    ],
    "return":"bool"
}
```

### 参数列表解析处理

值得注意的是在处理参数列表上，反射模块需要知道参数的类型，才能获取到method。

因此我们采用如下方式解析json。但当此参数类型，非基本数据类型时（object）时。

做了特殊处理，也可以通过传递的数据，反射组建此对象。

```
		  JSONArray prams = jsonObject.optJSONArray("params");
                if (prams != null) {
                    List<HashMap<Class<?>, Object>> pramsList = new ArrayList();
                    for (int i = 0; i < prams.length(); i++) {
                        JSONObject pram = new JSONObject(prams.get(i).toString());
                        Iterator<String> sIterator = pram.keys();
                        HashMap<Class<?>, Object> parmMap = null;
                        while (sIterator.hasNext()) {
                            parmMap = new HashMap<>();
                            String key = sIterator.next();
                            Class<?> typeClass = translateType(key);
                            parmMap.put(typeClass, translateParam(typeClass, pram.get(key)));

                        }
                        pramsList.add(parmMap);
                    }
                    testBean.setParams(pramsList);
                testBean.setReturnX(translateType(returnType));
            }
            testBean.setType(type);
```       
   
 **注意** 在转化class<?> 时做特殊处理。 这里是根据业务逻辑，获取了该获取的对象并返回。
   
```

    public Class<?> translateType(String key) {
        Class<?> pramsType = null;
        if (TextUtils.equals("bool", key)) {
            pramsType = boolean.class;
        } else if (TextUtils.equals("string", key)) {
            pramsType = String.class;
        } else if (TextUtils.equals("int", key)) {
            pramsType = int.class;
        } else if (TextUtils.equals("void", key)) {
            pramsType = void.class;
        } else if (TextUtils.equals("List", key)) {
            pramsType = List.class;
        } else if (TextUtils.equals("Bool", key)) {
            pramsType = Boolean.class;
        } else if (TextUtils.equals("callitem", key)) {
            pramsType = com.juphoon.cloud.JCCallItem.class;
        }
        return pramsType;
    }

    //处理特殊类型对象
    public Object translateParam(Class<?> type, Object value) {
        if (type == com.juphoon.cloud.JCCallItem.class) {
            List<JCCallItem> items = JCManager.getInstance().call.getCallItems();
            JCCallItem item = items.get(0);
            return item;
        }
        return value;
    }
```

## 反射模块

### 基本需求

1. 通过模块名(成员变量名称)，获取对象成员
2. 通过类名/模块名，方法名，参数名，参数类型，参数具体值，返回值类型，调用方法并获取返回值。
3. 兼容 空返回值，空参数。
4. 当参数/返回值类型非基本参数类型时。能根据参数，构造对象/获取对象，完成方法调用。（通过解析模块特殊处理）


### 获取对象 

1. 通过类名获取对象：

```
      Class<?> object = Class.forName(className);
```

2. 通过.class 方法获取对象，编译期就能检测的创建对象方式。优势是更加安全，在编译期可预知。此方法我们在编译期无法预知要测试的类，所以不可行。

```
   		jcManager.getClass().newInstance();
```

### 调用方法

1. 调用静态方法

```
 /**
     * 
     * @param T 返回值类型
     * @param className 类名  
     * @param methodName  方法名
     * @param pramsList 参数列表
     * @param <T>       返回值类型
     * @return
     * @throws NoSuchMethodException
     * @throws InvocationTargetException
     * @throws IllegalAccessException
     * @throws ClassNotFoundException
     */
    public static <T> T refStaticMethod(Class<T> T,String className, String methodName, ArrayList pramsList) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException, ClassNotFoundException {
        Class<?> cls = Class.forName(className);
        Class<?>[] pramsTypes = null;
        Object[] prams = null;
        if (pramsList != null) {
            pramsTypes = new Class[pramsList.size()];
            prams = new Object[pramsList.size()];
            for (int i = 0; i < pramsList.size(); i++) {
                pramsTypes[i] = pramsList.get(i).getClass();
                prams[i] = pramsList.get(i);
            }
        }

        Method method = cls.getMethod(methodName, pramsTypes);
        T returnValue;
        returnValue = (T) method.invoke(cls, prams);//第一个参数表示类的实例化对象(由于是静态方法)，第二个及其以后参数为可变参数

        return returnValue;
    }

```

2. 调用成员方法

***需要产生一个实例**

```
  /**
     * 调用成员函数
     * @param T
     * @param cls   成员对应的类名
     * @param obj   实例对象
     * @param methodName
     * @param pramsList
     * @param <T>
     * @return
     */

    public static <T> T refMethod(Class<T> T, Class<?> cls, Object obj, String methodName, List<HashMap<Class<?>, Object>> pramsList) throws ClassNotFoundException, NoSuchMethodException, InvocationTargetException, IllegalAccessException {
        Class<?>[] pramsTypes = new Class[pramsList.size()];
        Object[] prams = new Object[pramsList.size()];
        for (int i = 0; i < pramsList.size(); i++) {
            HashMap<Class<?>, Object> pramMap = pramsList.get(i);
            Iterator<Class<?>> iter = pramMap.keySet().iterator();
            while (iter.hasNext()) {
                Class<?> key = iter.next();
                Object value = pramMap.get(key);
                pramsTypes[i] = key;
                prams[i] = value;
            }
        }
        Method method = cls.getMethod(methodName, pramsTypes);
        T resultValue;//初始化返回值
        resultValue = (T) method.invoke(obj, prams);//第一个参数表示类的实例化对象，第二个及其以后参数为可变参数
        return resultValue;
    }

```

3. 获取对象成员

**getfield public field ，getDeclaredFields  与访问权限无关**

```
	/**
	*	className 类名
	* 	fieldName 成员变量名
	*
	**/
	
    public static   Field  refField(String className, String FieldName) throws ClassNotFoundException, NoSuchFieldException, IllegalAccessException {
        Class<?> cls = Class.forName(className);
        Field field = cls.getDeclaredField(FieldName);
        field.setAccessible(true);
        Class<?> type = field.getType();
        return  field;
    }

```

5. 将成员转化成实例对象

```
//次obj 为该成员所属的对象实例。
field.get(obj)
//需要field类型的成员变量，在此obj中确实已经实例化，否则receiver null 异常
//获取对象实际的类名
 Class<?> type = field.getType();

```

### 延伸部分
由于对于返回值的不确定，所以返回值也由范型处理

3. 范型处理返回值

```
public <T> T method(Class<T> T){
		...
		
		Object result=method.result();
	    T resultValue;//初始化返回值
       resultValue = (T)result;
       
       return resultValue;
}

``` 

4. 不定长参数可以传递数组

``` 
      @CallerSensitive
    public Method getMethod(String name, Class<?>... parameterTypes)
        throws NoSuchMethodException, SecurityException {
        return getMethod(name, parameterTypes, true);
    }
```
      
## 参考文章

[https://testerhome.com/topics/8880](https://testerhome.com/topics/8880)
[https://github.com/ximsfei/RefInject/blob/master/refinject/src/main/java/com/ximsfei/refinject/RefClass.java](https://github.com/ximsfei/RefInject/blob/master/refinject/src/main/java/com/ximsfei/refinject/RefClass.java)



