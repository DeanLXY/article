# Dalvik 可执行文件分包

## 背景

> 当您的应用及其引用的库达到特定大小时。字节码文件内的代码方法引用总数会达到上限（单个dex方法引用上限为65536=64K）

## 64K引用限制

> DEX规范将可在单个DEX文件内可引用的方法总数限制在65536
>
> 包括：android framework 方法/库方法/自己的方法

## 规避64K限制

> 您应该采取措施减少应用代码调用的引用总数，包括您的引用代码和包含的库定义的方法
>
> 1. 检查您的应用的直接和传递依赖项，不要为了使用几个实用的方法引入一个庞大的库
> 2. 通过ProGuard移除未使用的代码，apk中就不在包含未使用的方法

## 可执行文件分包支持

> 描述：默认情况下，Dalvik限制应用的每个APK只能使用单个的class.dex字节码文件
>
> 目标：一个dex文件方法引用限制为64k，可以使用多个dex文件扩展方法数限制

![2017-05-22_120002](.\media\2017-05-22_120002.png)



## 配置您的应用进行Dalvik可执行文件分包

1. 修改app/build.gradle文件启用可执行文件分包

```groovy
android {
    defaultConfig {
        ...
        minSdkVersion 15 
        targetSdkVersion 25
        multiDexEnabled true
    }
    ...
}

dependencies {
  compile 'com.android.support:multidex:1.0.1'
}
```

2. 替换或者修改Application

替换↓

```java
public class MyApplication extends MultiDexApplication { ... }
```

修改↓

```java
public class MyApplication extends SomeOtherApplication {
  @Override
  protected void attachBaseContext(Context base) {
     super.attachBaseContext(context);
     Multidex.install(this);
  }
}
```

2. 构建应用后，Android 构建工具会根据需要构建主 DEX 文件 (`classes.dex`) 和辅助 DEX 文件（`classes2.dex` 和 `classes3.dex` 等）。然后，构建系统会将所有 DEX 文件打包到您的 APK 中