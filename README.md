





# 防反编译利器技术框架

[TOC]

## 一、`APK`构建打包流程

### 1.1 打包流程总览

#### 1.1.1 打包流程图

 ![image](https://github.com/tianyalu/NePreventDecompilation/raw/master/show/apk_pack_process.png)

#### 1.1.2 工具列表

| 工具         | 描述                                                   |
| ------------ | ------------------------------------------------------ |
| `aapt/aapt2` | `Android`资源打包工具                                  |
| `aidl`       | `Android`接口描述语言转化为跨进程通讯`.java`文件的工具 |
| `javac`      | `Java`编译器                                           |
| `proguard`   | 代码混淆工具                                           |
| `dx/d8`      | 转化`.class`文件为`Davik VM`能识别的`.dex`文件         |
| `apkbuilder` | 打包生成`apk`包                                        |
| `jarsigner`  | 签名工具                                               |
| `zipalign`   | 字节码对齐优化工具                                     |

###  1.2 打包详细步骤

#### 1.2.1 资源合并

 ![image](https://github.com/tianyalu/NePreventDecompilation/raw/master/show/resource_combine.png)

#### 1.2.2 资源文件编译

资源文件可以分为以下类型：

* `res`资源
* `AndroidManifest`文件
* `assets`资源

 ![image](https://github.com/tianyalu/NePreventDecompilation/raw/master/show/resource_compile.png)

> 1. 生成`R.java`文件，赋予每一个非`assets`资源一个`ID`值，以常量的形式定义于`R.java`文件中；
> 2. 生成`resources.arsc`文件，用来描述那些具有`ID`值的资源的配置信息，它的内容就相当于是一个资源索引表，包含了所有的`id`值的数据集合。在该文件中，如果某个`id`对应的是`string`，那么该文件会直接包含该值；如果`id`对应的资源是某个`layout`或者`drawable`资源，那么该文件会存入对应资源的路径。

`R.java`结构：

```java
R.string.appname = 0x7f07006b;
// 7f: 编译的资源包
// 07: String类型
// 006b: appname 在String类型中出现的序号
```

`resources.arsc`文件结构（可以通过`Analyze apk`功能查看）：

 ![image](https://github.com/tianyalu/NePreventDecompilation/raw/master/show/resources_arsc.png)

#### 1.2.3 `aidl`文件编译

`aidl`: `Android Interface Definition Language`.

`aidl`工具解析接口定义文件，然后生成相应的`Java`代码接口供程序调用。如果在项目中没有用到`aidl`文件，则可以跳过这一步。

* 输入：`aidl`后缀的文件，位于工程`src/main/aidl`目录；
* 输出：可用于进程通讯的`C/S`端`java`代码，位于`build/generated/source/aidl`。

#### 1.2.4 `java`源码编译

`R.java`和`aidl`生成的`Java`文件，再加上工程的源代码，使用`javac`编译生成`class`文件。

* 输入：`java`源码文件（另外还包括了`aapt`生成的`R.java`、`aidl`生成的`java`文件，以及`BuildConfig.java`）;
* 输出：对于`gradle`编译，生成的`class`文件保存在`build/intermediates/classes`。

#### 1.2.5 `proguard`代码混淆

`javac`完成代码编译后，一般还会对源码进行混淆，类似于加密，目的是为了增加反编译的难度，同时也将代码名称进行缩短，减少代码占用体积。

* 输入：编译后的`.class`文件，混淆规则配置文件`proguard-rules.pro`；
* 输出：被混淆后的`.class`文件，混淆前后的映射文件。

#### 1.2.6 转换为`DEX`文件

`dx`工具生成可供`Android`系统虚拟机可以执行的`classes.dex`文件，`dx`会将`class`转换为`Dalvik`字节码，生成常量池，消除冗余数据等。

* 输入：所有的`.class`文件；
* 输出：`classes.dex`文件。

 ![image](https://github.com/tianyalu/NePreventDecompilation/raw/master/show/class_to_dex.png)

#### 1.2.7 打包`apk`文件

打包生成`apk`文件，旧的`apkbuilder`脚本已经废弃，现在通过`sdklib.jar`的`ApkBuilder`类进行打包。

* 输入：`.ap_`资源包文件，`classes.dex`文件，未编译的资源文件（`assets`资源等），`libs`等文件；
* 输出：`apk`文件。

#### 1.2.8 签名`apk`文件

对`apk`文件进行签名，签名后才能在设备上进行安装。

* 输入：上一步中生成的`.apk`文件、签名文件（`Debug or Release Keystore`）;
* 输出：签名后的`apk`文件。

 ![image](https://github.com/tianyalu/NePreventDecompilation/raw/master/show/apk_sign_file_meta_info.png)

#### 1.2.9 `zipalign`优化

`zipalign`对签名后的`apk`文件进行对齐处理，使`apk`中所有的资源文件距离文件起始偏移为4字节的整数倍，从而在通过内存映射访问`apk`文件时会更快，同时也减少了在设备上运行时的内存消耗。

* 输入：签名后的`apk`文件；
* 输出：对齐优化的`apk`文件。

#### 1.2.10 典型的`apk`文件

* `AndroidManifest.xml`：程序全局配置文件；
* `classes.dex`：`Dalvik`字节码；
* `resources.arsc`：资源索引表；
* `META-INF`：该目录下存放的是签名信息；
* `res`：该目录存放资源文件；
* `assets`：该目录可以存放一些配置或资源文件。

#### 1.2.11 关联技术

* `APK`加固；
* 资源混淆；
* 热修复；
* 插件化；
* 快速多渠道打包。

## 二、`DEX`编译过程与`MultiDex`方案原理

### 2.1 `DEX`编译过程

#### 3.2.1 编译流程

> 1. **`java`源码编译**：通过`javac`将源码编译为`.class`文件；
> 2. **多`DEX`分包**：脚本将类根据一定规则划分到主`dex`和从`dex`中，生成配置文件；
> 3. **`proguard`优化混淆**：对`.class`文件进行压缩、优化、混淆处理；
> 4. **转换为`DEX`文件**：`dx\d8`将`.class`文件转换为`dex`文件。

简化流程：

 ![image](https://github.com/tianyalu/NePreventDecompilation/raw/master/show/dex_compile_process.png)

#### 2.1.2 `DEX`文件格式简介

`DEX`格式结构：

| header     |
| ---------- |
| string_ids |
| type_ids   |
| proto_ids  |
| field_ids  |
| method_ids |
| class_def  |
| data       |
| link_data  |

 ![image](https://github.com/tianyalu/NePreventDecompilation/raw/master/show/dex_file_structure.png)

`010Editor` + `DEX`格式模板：

 ![image](https://github.com/tianyalu/NePreventDecompilation/raw/master/show/dex_file_structure2.png)

### 2.2 `MutiDex`方案背景

#### 2.2.1 背景

* 64K引用限制：

```java
Conversion to Dalivk format failed:
Unable to execute dex: method ID not in [0, 0xffff]: 65536
```

* 原因：

> 1. `DexOpt`优化的限制：早期`Android`系统中`DexOpt`会把每一个类的方法`id`检索起来，存在一个链表结构里面。链表的长度是用`short`类型来保存的，导致了方法`id`的数目不能够超过65536个；
> 2. `dalvik bytecode`的限制：`dalvik`的`invoke-kind`指令集，设置16bit表示方法引用数，最大值为65536 `invoke-kind{vC, vD, vE, vF, vG}, meth@BBBB`。

* `Android 5.0`之前：使用`Dalvik`可执行文件分包支持库；
* `Android 5.0`及其之后版本：

> `Android 5.0`(`API`级别21)及更高版本使用`ART`运行时，后者原生支持从`APK`文件加载多个`DEX`文件，`ART`在应用安装时执行预编译，扫描`classesN.dex`文件，并将它们编译成单个`.oat`文件，供`Android`设备执行。因此如果`minSdkVersion`为21或更高值，则不需要`Dalvik`可执行文件分包支持库。

#### 2.2.2 `MultiDex`方案配置

修改`build.gradle`文件：

```groovy
//minSdkVersion设置为21或者更高值
android {
  defaultConfig{
    //...
    minSdkVersion 21
    targetSdkVersion 28
    multiDexEnabled true
  }
  //...
}
```

```groovy
//minSdkVersion设置低于21时
android {
  defaultConfig{
    //...
    minSdkVersion 15
    targetSdkVersion 28
    multiDexEnabled true
  }
  //...
  dependencies {
    compile 'com.android.support:multidex:1.0.3'
  }
}
```

修改`Application`代码：

```java
//继承MultiDexApplication
public class MyApplication extends MultiDexApplication {
  //...
}
```

或者：

```java
//调用MultiDex.install(this) 启用分包
public class MyApplication extends SuperApplication {
  @Override
  protected void attachBaseContext(Context base) {
    super.attachBaseContext(base);
    MultiDex.install(this);
  }
}
```

#### 2.2.3 分包流程

* 生成`manifest_keep.txt`：解析出`AndroidManifest.xml`中所有的组件类：包括`Activity`、`Service`、`Receiver`以及`ContentProvider`，这些类将和`Application`入口类一起放在`manifext_keep.txt`文件中；
* 生成`maindexlist.txt`文件：查找`manifest_keep.txt`中所有类的直接引用类，将其保存在`maindexlist.txt`;
* 生成多`dex`：`dx`工具接收参数，将`maindexlist.txt`文件包含的所有`class`编译进主`dex`，并生成其它`dex`.

#### 2.2.4 `MultiDex.install`执行流程

 ![image](https://github.com/tianyalu/NePreventDecompilation/raw/master/show/multidex_install_execute_sequence.png)

#### 2.2.5 `MultiDex`加载原理

 ![image](https://github.com/tianyalu/NePreventDecompilation/raw/master/show/multidex_install_src_code1.png)

![image](https://github.com/tianyalu/NePreventDecompilation/raw/master/show/multidex_install_src_code2.png)

## 三、`proguard`混淆与防反编译

### 3.1 代码混淆

#### 3.1.1 混淆流程

![image](https://github.com/tianyalu/NePreventDecompilation/raw/master/show/proguard_process.png)

#### 3.1.2 `proguard`配置

`build.gradle`文件：

```groovy
android {
  compileSdkVersion 23
  buildToolsVersion "24.0.1"
  defaultConfig {
    //...
  }
  buildTypes {
    release {
      //主要看这里
      zipAlignEnabled true
      minifyEnabled true
      proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
    }
  }
}
```

#### 3.1.3 常见混淆配置

* `keep`用来保留`Java`的元素不进行混淆：

| 关键字                    | 释义                                                         |
| ------------------------- | ------------------------------------------------------------ |
| `-Keep`                   | 保留类和类中的成员，防止它们被混淆或移除                     |
| `-Keepclassmembers`       | 只保留类中的成员，防止它们被混淆或者移除                     |
| `-KeepclassesWithmembers` | 保留类和类中的成员，防止它们被混淆或移除，前提是指名的类中的成员<br/>必须存在，如不存在还是会混淆 |

* `dontwarn`：引入的`library`可能存在一些无法找到的引用和其它问题，在`build`时可能会发出警告会导致`build`终止，因此未来保证`build`继续，需要使用`dontwarn`处理这些无法解决的`library`的警告。

```java
-keep public class com.sty.widget.**
-keepclassmembers public class * extends android.view.View {
  void set*(***);
  *** get*();
}
-keepclassmembers class **.R$* {
  public static <fields>;
}
-keepclasseswithmembernames class * {
  native <methods>;
}
-dontwarn android.support.**
```

#### 3.1.4 需要`keep`的情况

* `enum`枚举
* `Android`四大组件类
* 自定义控件，继承`View`的类
* 实现了`android.os.Parcelable`接口
* 序列化和反序列化的类
* 反射的成员变量或者方法（包括`jni`、`js`中的反射）
* 注解
* `Native`方法
* 三方`sdk`

#### 3.1.5 字符串混淆

* 字符串编码混淆：编码混淆就是先将字符串转换成16进制的数组或者`Unicode`编码，在使用的时候才能恢复成字符串。破解者看到之后是一串数字或者乱码，难以分析。

```java
private String encodeString() {
  byte[] strBytes = {0x48, 0x65, 0x6c, 0x6f, 0x20, 0x77, 0x6f, 0x72, 0x6c, 0x64};
  String str = new String(strBytes);
  return str;
}
```

* 字符串加密：加密处理就是实现在本地将字符串加密，将密文硬编码到源程序中，再实现一个解密函数，在引用密文的地方调用解密函数解密。

### 3.2 防反编译

#### 3.2.1 花指令

在原始程序中插入一组无用的字节，但有不会改变程序的元素逻辑，程序仍然可以正常运行，然而反编译工具在反编译这些字节时会出错，造成反汇编工具失效，提高破解难度。例如下面的`dalvik`指令：

![image](https://github.com/tianyalu/NePreventDecompilation/raw/master/show/junk_code1.png)

如果反编译工具采用线性扫描算法，会错误识别花指令导致出错：

![image](https://github.com/tianyalu/NePreventDecompilation/raw/master/show/junk_code2.png)

#### 3.2.2 资源混淆

* 修改`aapt`：修改`aapt`处理资源文件相关的源码，参考`proguard`方式对`apk`中资源文件名使用简短无意义名称进行替换，给破解者制造困难，从而做到资源的相对安全。

![image](https://github.com/tianyalu/NePreventDecompilation/raw/master/show/appt_proguard.png)

* 修改`resources.arsc`：根据`resources.arsc`文件格式，修改资源名与路径的映射关系。

| 工具                               | 描述                                                         |
| ---------------------------------- | ------------------------------------------------------------ |
| `table stringblock`                | 改变文件指向路径，例如：`res/layout/test.xml`，改为`res/layout/a.xml` |
| 资源文件名                         | 修改资源的文件名，即将`test.xml`重命名为`a.xml`              |
| `specsname stringblock`            | 旧的`specsname`除了白名单部分全部废弃，替换成所有混淆方案中用到的字符，由于重复使用`[a-z0-9]`，`specsname`的总数量会大大减少 |
| `entry`中指向的`specsname`中的`id` | 例如原本`test.xml`它指向`specsname`中的第十项，我们需要用混淆后的`a`项的位置改写 |
| `table chunk`的大小                | 修改`table chunk`的最后大小                                  |

