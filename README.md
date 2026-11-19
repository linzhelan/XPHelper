# XPHelper

[![JitPack](https://jitpack.io/v/suzhelan/XpHelper.svg)](https://jitpack.io/#suzhelan/XpHelper)

用于开发 Android Xposed 模块，提供反射工具、DexKit 查询与缓存、模块 Activity 与资源加载，以及基于 FastKV
的配置存储。

[接入](#接入) · [反射](#反射) · [DexKit](#dexkit) · [配置存储](#配置存储) · [Activity 与资源](#activity-与资源) · [Kotlin 扩展](#kotlin-扩展) · [底层工具](#底层工具) · [常见问题](#常见问题)

如果你已经知道类名和成员签名，可以直接用[反射](#反射)。如果成员名随版本变化，可以用 [DexKit](#dexkit) 根据字符串、参数类型等特征定位成员，再用反射调用。保存模块开关和用户设置可以用[配置存储](#配置存储)，需要在宿主进程中显示模块设置页时，参考 [Activity 与资源](#activity-与资源)。

## 接入

本仓库的 `:xphelper` 模块配置为 minSdk 24，构建使用 JDK 17。使用发布版时，以所选版本的构建配置为准。

### 1. 添加依赖

在 `settings.gradle.kts` 的 `dependencyResolutionManagement.repositories` 中加入以下仓库。

```kotlin
maven { url = uri("https://jitpack.io") }
maven { url = uri("https://api.xposed.info/") }
```

在模块的 `build.gradle.kts` 中加入依赖，并将 `<version>` 替换为 JitPack 上可用的版本。本仓库内可使用
`implementation(project(":xphelper"))`。使用 `BaseActivity` 时在模块中引入 AndroidX AppCompat，使用
`BaseComposeActivity` 时引入 AndroidX Activity 与所需的 Compose
依赖。[JitPack 版本列表](https://jitpack.io/#suzhelan/XpHelper)

```kotlin
dependencies {
    implementation("com.github.suzhelan:XpHelper:<version>")
    compileOnly("de.robv.android.xposed:api:82")
}
```

### 2. 声明 Xposed 入口

以下使用本仓库演示的传统 Xposed 入口。在模块的 `AndroidManifest.xml` 的 `<application>` 下声明元数据，在 `src/main/assets/xposed_init` 中写入入口类的完整类名，每行一个。本例写 `com.example.module.ModuleEntry`。已有 Xposed 入口的项目可以将下一节的初始化接到自己的回调中。

```xml
<meta-data android:name="xposedmodule" android:value="true" />
<meta-data android:name="xposeddescription" android:value="XPHelper 演示模块" />
<meta-data android:name="xposedminversion" android:value="82" />
```

### 3. 在宿主进程初始化

这里把自己的测试应用 `com.example.demo` 当作宿主，也就是模块运行时所处的目标应用。测试应用的清单需声明至少一个 Activity。入口只处理示例宿主的主进程，并在 `Application.attach` 完成后拿到 Application。模块自身的 Application Context 和宿主 Context 各自对应不同的类加载器、APK 和存储目录。

```java
package com.example.module;

import android.app.Application;
import android.content.Context;

import de.robv.android.xposed.IXposedHookLoadPackage;
import de.robv.android.xposed.IXposedHookZygoteInit;
import de.robv.android.xposed.XC_MethodHook;
import de.robv.android.xposed.XposedBridge;
import de.robv.android.xposed.XposedHelpers;
import de.robv.android.xposed.callbacks.XC_LoadPackage;
import top.sacz.xphelper.XpHelper;

public class ModuleEntry implements IXposedHookZygoteInit, IXposedHookLoadPackage {
    private static final String HOST = "com.example.demo";

    @Override
    public void initZygote(StartupParam param) {
        XpHelper.initZygote(param);
    }

    @Override
    public void handleLoadPackage(XC_LoadPackage.LoadPackageParam param) {
        if (!HOST.equals(param.packageName) || !HOST.equals(param.processName)) {
            return;
        }

        XposedHelpers.findAndHookMethod(
                Application.class, "attach", Context.class, new XC_MethodHook() {
                    @Override
                    protected void afterHookedMethod(MethodHookParam hook) {
                        Application host = (Application) hook.thisObject;
                        XpHelper.initContext(host);
                        XposedBridge.log("XPHelper initialized for " + host.getPackageName());
                    }
                });
    }
}
```

安装模块后，在 Xposed 环境中启用模块并选择测试应用作为作用域，再重启测试应用进程。日志中出现 `XPHelper initialized for com.example.demo` 后，就可以在初始化后的模块逻辑里使用下文的 API。

### 初始化接口速查

| `XpHelper` 方法 | 调用时机与用途 |
| --- | --- |
| `initZygote(startupParam)` | 在 Xposed 的同名回调中保存模块 APK 路径。 |
| `initContext(application)` | 在目标应用进程拿到宿主的 Application Context 后调用。设置宿主类加载器、配置目录、Activity 代理，并按宿主版本检查 DexKit 缓存。 |
| `setConfigPath(pathDir)` | 为后续创建的配置实例指定可写目录。若使用 `initContext`，在其后设置目录，因为它会重设默认目录。 |
| `setConfigPassword(password)` | 为后续创建的配置实例设置全局密码及加密开关。 |
| `injectResourcesToContext(context)` | 在 `initZygote` 后，把模块 APK 的资源加载到指定 Context。 |

`context`、`classLoader` 和 `moduleApkPath` 分别保存宿主上下文、宿主原始类加载器和模块 APK
路径。完整入口见 [InjectHook.java](app/src/main/java/top/sacz/hook/InjectHook.java)。

## 反射

反射可以在运行时获取类、方法、字段和构造器。下面在测试应用中添加一个 Java 类，供本节和后续的 DexKit 示例使用。

```java
package com.example.demo;

public class DemoSettings {
    private String title = "初始名称";

    public DemoSettings() {}

    public DemoSettings(String title) {
        this.title = title;
    }

    private String rename(String title) {
        this.title = title;
        return displayName();
    }

    private String displayName() {
        return "XPHELPER_DEMO_TITLE: " + title;
    }

    public static String version() {
        return "1";
    }
}
```

### 创建实例、调用方法、读写字段

即使模块编译时无法引用测试应用的类，也可以在初始化后通过宿主加载器按类名获取它。确保示例类进入最终 APK，例如在宿主的调试代码中引用该类。

```java
import top.sacz.xphelper.reflect.ClassUtils;
import top.sacz.xphelper.reflect.ConstructorUtils;
import top.sacz.xphelper.reflect.FieldUtils;
import top.sacz.xphelper.reflect.MethodUtils;
import top.sacz.xphelper.util.DexMethodDescriptor;

// 放在支持抛出 ReflectiveOperationException 的初始化后逻辑中
Class<?> type = ClassUtils.findClass("com.example.demo.DemoSettings");
Object settings = new DexMethodDescriptor("Lcom/example/demo/DemoSettings;-><init>()V")
        .getConstructorInstance(ClassUtils.getClassLoader())
        .newInstance();
Object namedSettings = ConstructorUtils.create(type)
        .paramTypes(String.class)
        .newFirstInstance("我的设置");

String label = MethodUtils.create(type)
        .methodName("rename")
        .params(String.class)
        .returnType(String.class)
        .invokeFirst(settings, "XPHelper");

String title = FieldUtils.getField(settings, "title", String.class);
FieldUtils.setField(settings, "title", "新名称");

String version = MethodUtils.create(type)
        .methodName("version")
        .params()
        .invokeFirstStatic();
```

`params(String.class)` 用来选中声明为 `rename(String)` 的重载，`invokeFirst(settings, "XPHelper")` 才是实际调用。上面的 `label` 为 `XPHELPER_DEMO_TITLE: XPHelper`，读取到的 `title` 为 `XPHelper`。静态方法由类持有，所以 `invokeFirstStatic()` 只接收方法实参。

### 精确匹配和处理空结果

- 同名重载优先用 `params(...)` 或 `paramTypes(...)` 指定声明中的参数类型。实参为 `null` 时仍可显式指定声明类型。需要忽略某个参数位置的类型时，传 `Ignore.class`。
- 需要查看有几项匹配时，先用 `getResult()` 取列表。反射枚举顺序不保证稳定，定位唯一成员时应补足名称和类型条件。
- 可选功能使用 `firstOrNull()` 或 `lastOrNull()`，必需成员可用 `first()` 或 `last()`，无结果时会抛 `ReflectException`。取结果时会先查当前类，当前类无结果才查父类。
- 条件设置完成后再取结果。查另一组成员时重新调用 `create(...)`，因为反射查找器会保留已经得到的结果。

### 反射 API 速查

#### `ClassUtils`

- `intiClassLoader(loader)` 单独初始化反射类加载器。
- `getClassLoader()` 取支持缓存和类型描述符的宿主加载器，`getModuleClassLoader()` 取模块加载器。
- `findClass(name)` 支持类名、DEX 类描述符及基本类型，失败抛异常。
- `findClassOrNull(name)` 在查找失败时返回 `null`。
- `isCommonlyUsedClass(name)` 检查是否属于内置的常见包名前缀。

#### `MethodUtils`

- `create(obj / clazz / className)` 创建查找器。
- `methodName(name)`、`returnType(type)`、`params(types...)`、`paramCount(count)` 分别筛选名称、返回类型、参数类型与个数。
- `invokeFirst(obj, args...)`、`invokeLast(obj, args...)` 调用首项或末项，`invokeNoReturn(obj, args...)` 忽略首项的返回值，`invokeFirstStatic(args...)` 调用首个匹配项作为静态方法。
- 旧方法 `callFirst`、`callLast`、`callFirstStatic` 分别对应上述同名 `invoke` 方法。
- `getDescriptor(method)` 生成 DEX 描述符，`getMethodByDescriptor(desc)` 或 `getMethodByDescriptor(desc, loader)` 解析为可访问的 `Method`。

#### `FieldUtils`

- `create(obj / clazz / className)` 创建查找器。
- `fieldName(name)` 和 `fieldType(type)` 筛选名称和类型。
- `firstValue(obj)`、`lastValue(obj)` 读取首项或末项，`setFirst(obj, value)`、`setLast(obj, value)` 写入首项或末项。
- `getField(obj, name, type)` 按名称和类型读取，`setField(obj, name, value)` 按名称写入，`getFirstType(obj, type)` 按类型读取。
- `setFirstType(obj, value)` 当前以值的类名筛选字段名，按类型写入可用 `create(obj).fieldType(type).setFirst(obj, value)`。
- `getDescriptor(field)` 和 `getFieldByDescriptor(desc[, loader])` 在字段和描述符之间转换。

#### `ConstructorUtils`

- `create(obj / clazz / className)` 创建查找器。
- `paramCount(count)` 筛选非零参数个数。当前实现以 `0` 表示跳过数量过滤，类同时有有参构造器时，`paramTypes()` 可能触发数组越界。要精确获取无参构造器，可用上面的 `<init>()V` 描述符或 Java 的 `clazz.getDeclaredConstructor()`。
- `paramTypes(types...)` 筛选参数类型，`newFirstInstance(args...)` 调用首个匹配的构造器。
- `newInstance(clazz, types, args...)` 显式指定类型，`newInstance(clazz, args...)` 根据非空实参推断类型。

#### 三个反射查找器共有

- `matchParentClass(true)` 启用类型的双向可赋值匹配。
- 参数位置可传 `Ignore.class` 作为通配，字段类型也可传 `Ignore.class`。
- 调用静态方法或读写静态字段时，选择对应成员并传 `null` 作为实例。

#### `BaseFinder`

- `find()` 扫描当前类的声明成员。
- `findAndSuper()` 在无结果时向父类查找并将结果设为可访问。
- `getResult()` 返回列表，`get(index)` 按从零开始的下标取项，`first()`、`last()` 取首尾，无结果时抛 `ReflectException`，`firstOrNull()`、`lastOrNull()` 无结果时返回 `null`。
- 取结果的方法会自动触发查找。
- `buildSign()` 返回缓存签名。
- 直接调用 `ConstructorUtils.find()` 时，缓存未命中会返回 `null`，取结果请使用 `getResult()` 等方法。

## DexKit

DexKit 扫描 APK 里的 DEX 字节码。反射适合用已知的类名和签名找成员，DexKit 还能利用方法内的字符串、数字、读写字段和调用关系做筛选。查到的是 `Method`、`Field`、`Class` 或 `Constructor`，可以继续读取元数据或像普通反射对象一样使用。

### 按签名和字节码特征查找

初始化后可直接查询宿主 APK。下面按声明类和方法名查找，查不到时返回 `null`。

```kotlin
import top.sacz.xphelper.dexkit.ClassFinder
import top.sacz.xphelper.dexkit.DexFinder
import top.sacz.xphelper.dexkit.FieldFinder
import top.sacz.xphelper.dexkit.MethodFinder
import top.sacz.xphelper.reflect.ClassUtils

val demoClass = ClassUtils.findClass("com.example.demo.DemoSettings")
val method = MethodFinder.build()
    .declaredClass(demoClass)
    .methodName("rename")
    .paramCount(1)
    .firstOrNull()

val field = DexFinder.findField {
    declaredClass = demoClass
    fieldName = "title"
}.firstOrNull()

// 利用方法中的字符串和返回类型查找，省略方法名
val displayName = MethodFinder.build()
    .searchPackages("com.example.demo")
    .declaredClass(demoClass)
    .usedString("XPHELPER_DEMO_TITLE")
    .returnType(String::class.java)
    .paramCount(0)
    .firstOrNull()

// 找到使用 title 的方法
val titleUsers = MethodFinder.build()
    .declaredClass(demoClass)
    .usedFields(FieldFinder.build().declaredClass(demoClass).fieldName("title"))
    .find()

// 找包含 title 字段的类，以及 String 参数的构造器
val owner = ClassFinder.build()
    .searchPackages("com.example.demo")
    .fields(FieldFinder.build().fieldName("title").fieldType(String::class.java))
    .firstOrNull()

val constructor = MethodFinder.build()
    .declaredClass(demoClass)
    .parameters(String::class.java)
    .firstConstructorOrNull()
```

这里用没有开启混淆和优化的宿主调试构建验证条件。实际使用时，可先用 `find()` 检查候选数量，再加入参数、常量或成员关系缩小范围。 `usedFields(...)` 匹配使用字段的方法，`FieldFinder.readMethods(...)` 和 `writeMethods(...)` 则从字段一侧指定读写它的方法。

`invokeMethods(callee)` 查找调用了 `callee` 的方法。反过来，`callMethods(caller)` 查找被 `caller` 调用的方法。例如上面的 `rename` 调用 `displayName`，两种条件各自对应其中一端。

### 自定义查询与缓存

Finder 的 `find()` 和取首项方法会自动查缓存。更复杂的条件可以通过原生 Bridge 查询，再由 `queryMethod` 等接口缓存。下例用宿主加载器将 DexKit 元数据转成可调用的 Java 反射对象。

```kotlin
import org.luckypray.dexkit.query.FindMethod
import org.luckypray.dexkit.query.matchers.MethodMatcher
import top.sacz.xphelper.dexkit.DexFinder
import top.sacz.xphelper.reflect.ClassUtils

val methods = DexFinder.queryMethod("demo:displayName:v1") { bridge ->
    val request = FindMethod.create().matcher(
        MethodMatcher.create()
            .declaredClass("com.example.demo.DemoSettings")
            .name("displayName")
    )
    bridge.findMethod(request).map { data ->
        data.getMethodInstance(ClassUtils.getClassLoader()).apply {
            isAccessible = true
        }
    }
}
```

- 自定义查询调整条件后，更新缓存键。例如将 `demo:displayName:v1` 改成 `demo:displayName:v2`。空列表也会被缓存，`existCache()` 为 `true` 只表示已有记录。
- Finder 缓存会在初始化时按宿主版本检查。开发期间想重新执行查询，可调用 `DexFinder.clearCache()`，它会清除全部 DexKit 查询记录。
- 默认 Bridge 在最近一次获取后的 10 秒自动关闭，长查询开始前可调用 `setAutoCloseTime(0)`，用完后调用 `close()`。`close()` 释放 Bridge，已保存的查询记录仍可读取。
- 直接使用 Bridge 或 Matcher 时，在模块中引入与 XPHelper 版本匹配的 DexKit。本仓库当前使用 `org.luckypray:dexkit:2.2.0`。耗时扫描可放到工作线程执行。

### DexKit API 速查

#### `DexFinder`

- `create(apkPath)` 在 Bridge 未创建时打开指定 APK。
- `getDexKitBridge()` 获取 Bridge，首次按需打开宿主 APK，并重置自动释放计时。
- `setAutoCloseTime(ms)` 设置计时，默认 10 秒，传 `0` 后续获取时不再安排自动关闭。
- `close()` 释放 Bridge 并取消计时。
- `clearCache()` 清空持久查询缓存。
- `isLoadLibrary` 是库加载时使用的原子标记。

#### Kotlin DSL

- `DexFinder.findMethod { … }`、`findField { … }`、`findClass { … }` 分别配置 `MethodInfo`、`FieldInfo`、`ClassInfo` 并返回对应 Finder。
- 独立创建这些配置对象时，可设置与 Finder 条件对应的属性，再调用 `generate()`。
- 类的接口条件在 DSL 中使用 `interfaces`，修饰符使用 `modifiers` 和 `matchType`。

#### 自定义缓存查询

- `DexFinder.queryMethod(key, block)`、`queryField(key, block)`、`queryClass(key, block)`、`queryConstructor(key, block)` 仅在缓存未命中时把 Bridge 传给回调。
- 回调返回对应的反射对象列表。
- 为不同结果类型与不同查询条件使用不同的键。

#### 三个 Finder 共有

- `build()` 创建空查询，`from(method / field / clazz)` 用已有反射对象填充条件，其中 `ClassFinder.from(clazz)` 需要 `clazz` 有父类。
- `modifiers(flags, matchType)` 按修饰符匹配，例如 `Modifier.PUBLIC` 配合 `MatchType.Contains`。
- `searchPackages(packages...)` 和 `excludePackages(packages...)` 限定或排除包。
- `find()` 返回全部结果，`first()` 无结果时抛异常，`firstOrNull()` 无结果返回 `null`。
- `existCache()` 检查查询键是否存在，`toString()` 返回作为缓存键的查询描述。

#### `MethodFinder`

- `declaredClass(clazz)`、`methodName(name)`、`returnType(type)`、`parameters(types...)`、`paramCount(count)` 匹配方法签名。
- 零参用 `paramCount(0)`。
- `usedString(strings...)`、`usingNumbers(numbers...)` 匹配使用的常量。
- `usedFields(fields...)` 接收 `Field` 或 `FieldFinder`。
- `invokeMethods(methods...)` 匹配被调用的方法，`callMethods(methods...)` 匹配调用方。
- `findConstructor()` 返回构造器列表，`firstConstructor()` 和 `firstConstructorOrNull()` 分别在无结果时抛异常或返回 `null`。
- `buildMethodMatcher()` 将当前条件转换为 DexKit Matcher，`toMethodMatcher(method)` 直接从反射对象构造 Matcher。

#### `FieldFinder`

- `declaredClass(clazz)`、`fieldName(name)`、`fieldType(type)` 匹配字段。
- `readMethods(methods...)` 和 `writeMethods(methods...)` 匹配读写该字段的方法，接收 `Method` 或 `MethodFinder`。
- `buildFieldMatcher()` 转换当前条件，`toFieldMatcher(field)` 直接从反射对象构造 Matcher。

#### `ClassFinder`

- `className(name)`、`superClass(name)`、`addInterface(names...)` 匹配类名、父类与接口。
- `usedString(strings...)` 匹配类中使用的字符串。
- `fields(fieldFinders...)` 和 `methods(methodFinders...)` 匹配类中声明的成员。
- `buildClassMatcher()` 将类条件转换为 DexKit Matcher。

Java 可直接使用 `MethodFinder.build()`、`FieldFinder.build()`、`ClassFinder.build()`。调用 `DexFinder` 的普通方法时使用 `DexFinder.INSTANCE`，例如 `DexFinder.INSTANCE.close()`。 `getDexKitBridge()` 和四个 `query*` 方法可直接静态调用。

## 配置存储

### 初始化并选择存储

可单独调用 `ConfigUtils.initialize(context)` 或 `initialize(path)` 后使用。Context 版本将目录设为
`filesDir/XpHelper`。`ConfigUtils()` 使用名为 `default` 的存储，`ConfigUtils(key)` 指定名称，
`ConfigUtils(key, password)` 指定实例密码。先调用 `setGlobalPassword(password)` 或
`XpHelper.setConfigPassword(password)` 设置非空全局密码来启用加密。全局密码为空时，传入实例密码仍会创建未加密的存储。FastKV
加密适配器会原样处理整数和长整型。

在 Xposed 回调中，`XpHelper.initContext(host)` 已完成目录初始化。独立在模块应用中使用时，可以在该应用自己的 `Application.onCreate()` 中调用 `ConfigUtils.initialize(this)`。宿主进程和模块应用的 `filesDir` 各自属于不同的应用，使用相同的存储名不会让两边共享配置。

### 保存开关、列表和对象

```kotlin
import top.sacz.xphelper.util.ConfigUtils

val config = ConfigUtils("settings")
config.put("enabled", true)
val enabled = config.getBoolean("enabled", false)
config.putStringList("tags", listOf("demo"))
val tags = config.getStringList("tags")
```

基本类型读写按类型配对，例如 `put("retry", 3)` 用 `getInt("retry", 0)` 读取。想区分“从未保存”和“保存了默认值”，先用 `containsKey("retry")` 判断。

对象存储使用 kotlinx.serialization。若对象类型定义在你的模块里，在模块中应用与 Kotlin 版本一致的 `org.jetbrains.kotlin.plugin.serialization` 插件，并加入 kotlinx.serialization 依赖。本仓库的示例使用 `org.jetbrains.kotlinx:kotlinx-serialization-json:1.11.0`。

```kotlin
import kotlinx.serialization.Serializable
import top.sacz.xphelper.util.ConfigUtils

@Serializable
data class Profile(val name: String, val showTips: Boolean = true)

fun saveProfile(config: ConfigUtils) {
    val profile = Profile("Demo")
    config.putObject("profile", profile)
    config.putList("history", mutableListOf(profile))

    val current = config.getObject<Profile>("profile")
    val history = config.getList<Profile>("history")
}
```

Java 使用显式的类型参数。以下假设 `Profile` 是上面定义的可序列化类型。

```java
import java.util.ArrayList;
import java.util.List;
import top.sacz.xphelper.util.ConfigUtils;

ConfigUtils config = new ConfigUtils("settings");
Profile profile = new Profile("Demo", true);
config.putObject("profile", profile, Profile.class);
Profile current = config.getObject("profile", Profile.class);

List<Profile> history = new ArrayList<>();
history.add(profile);
config.putList("history", history, Profile.class);
List<Profile> storedHistory = config.getList("history", Profile.class);
```

### 配置 API 速查

| 方法 | 用途 |
| --- | --- |
| `put(key, value)` | 写入字符串、基本数值、布尔值、字节数组，或将列表、Map 与可序列化对象写为 JSON。 |
| `getInt`、`getDouble`、`getLong`、`getBoolean`、`getFloat`、`getBytes`、`getString` | 按 `(key, def)` 读取对应类型。Kotlin 可省略 `def`，默认依次为 `0`、`0.0`、`0L`、`false`、`0f`、空字节数组、空字符串。Java 须传默认值。 |
| `putStringList(key, list)`、`getStringList(key)` | 读写字符串列表，缺失时返回空列表。 |
| `putObject(key, value)`、`getObject<T>(key)` | 使用 kotlinx.serialization 读写对象，缺失时返回 `null`。自定义类型可用 `@Serializable` 声明。重载可在末尾传 `KSerializer<T>` 或 `Class<T>`，Java 可使用 `config.getObject("profile", Profile.class)`。 |
| `putList(key, mutableList)`、`getList<T>(key)` | 读写对象列表，缺失时返回空的可变列表。重载可在末尾传元素的 `KSerializer<T>` 或 `Class<T>`。 |
| `putAll(map)`、`toMap()` | 调用 FastKV 批量写入，或取出底层存储值。以 JSON 保存的对象在 Map 中为字符串。 |
| `getAllKeys()`、`containsKey(key)`、`remove(key)`、`clearAll()` | 列出键、检查键、删除单项和清空当前存储。 |

## Activity 与资源

### 创建模块页面

模块页面可以在宿主进程中运行。以 Compose 为例，在模块中增加以下类。 `super.onCreate(savedInstanceState)` 会加载模块资源，之后可以设置页面内容。

```kotlin
import android.os.Bundle
import androidx.activity.compose.setContent
import androidx.compose.material3.Text
import top.sacz.xphelper.activity.BaseComposeActivity

class ModuleActivity : BaseComposeActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            Text("XPHelper 模块设置")
        }
    }
}
```

使用 XML 布局时，可以继承 `BaseActivity`，同样先调用 `super.onCreate(savedInstanceState)`，再调用 `setContentView(R.layout.activity_settings)`。这里的 `R` 来自模块，`activity_settings.xml` 放在模块的 `res/layout` 下。

### 从宿主界面打开

在 `initContext` 完成后，从宿主已有页面的点击事件调用下面的函数。传入的 `hostActivity` 是宿主页面实例。初始化已安装 Activity 代理，它使用宿主清单中的 Activity 启动模块页面。

```kotlin
import android.app.Activity
import android.content.Intent
import top.sacz.xphelper.util.ActivityTools

fun openSettings(hostActivity: Activity) {
    ActivityTools.runOnUiThread {
        val intent = Intent(hostActivity, ModuleActivity::class.java)
            .putExtra("source", "host_settings")
        hostActivity.startActivity(intent)
    }
}
```

模块页面可用 `intent.getStringExtra("source")` 读取参数。已有宿主 Activity 实例时可以直接传递。需要查找当前页面时，可以调用 `ActivityTools.getTopActivity()`，它可能返回 `null`。

### 在宿主页面中使用模块资源

如果只想在宿主页面显示模块的文字或布局，可以在主线程先把模块 APK 的资源加载到宿主 Context，再使用模块的资源 ID。以下假设模块定义了 `res/values/strings.xml` 中的 `module_title`。

```kotlin
import android.app.Activity
import android.widget.Toast
import top.sacz.xphelper.XpHelper
import top.sacz.xphelper.util.ActivityTools

fun showModuleTitle(hostActivity: Activity) {
    ActivityTools.runOnUiThread {
        XpHelper.injectResourcesToContext(hostActivity)
        val title = hostActivity.getString(R.string.module_title)
        Toast.makeText(hostActivity, title, Toast.LENGTH_SHORT).show()
    }
}
```

宿主和模块的资源 ID 可能冲突。本仓库的演示模块用 AAPT 参数将模块资源包 ID 配置为 `0xf2`，可参考它的构建配置。完整的启动和 Compose 页面见以下示例。[app/build.gradle.kts](app/build.gradle.kts) · [ActivityTest.kt](app/src/main/java/top/sacz/hook/ActivityTest.kt) · [ModuleActivity.kt](app/src/main/java/top/sacz/hook/activity/ModuleActivity.kt)

### Activity API 速查

#### `ActivityProxyManager`

- `initActivityProxyManager(hostContext)` 初始化代理，由 `XpHelper.initContext` 自动调用。
- 默认使用宿主清单中的首个 Activity，宿主需至少声明一个 Activity。
- `isModuleActivity(className)` 检查类是否继承两种基类之一。
- `HostActivityClassName` 保存默认代理类名，`ACTIVITY_PROXY_INTENT` 保存代理使用的 Intent 键。

#### `BaseActivity`、`BaseComposeActivity`

- `getClassLoader()` 返回页面使用的组合加载器。
- 基类的 `onRestoreInstanceState(state)` 会为视图状态设置加载器，再交还系统恢复。
- `BaseActivity` 子类可调用受保护的 `requestTranslucentStatusBar()` 设置透明系统栏。
- `BaseComposeActivity.BaseActivityClassLoader(moduleLoader)` 创建组合加载器，其 `loadClass(name, resolve)` 按包选择系统、宿主或模块加载器。

#### `ActivityTools`

- `getAllActivity(context)` 读取该应用清单中的 Activity 信息。
- `getAllChildViews(activity / rootView)` 递归列出子 View。
- `getTopActivity()` 查找首个未暂停的 Activity，找不到时返回 `null`。
- `runOnUiThread(task)` 在主线程执行任务，`runOnUiThreadDelay(ms, task)` 延迟执行。
- `injectResourcesToContext(context, apkPath)` 加载指定 APK 的资源，Android 11 及以上从后台线程调用时会把资源挂载提交到主线程。

## Kotlin 扩展

### 用扩展函数调用和读写

前面的 Java 反射示例也可以用 Kotlin 扩展来写。这里继续使用测试应用的 `DemoSettings`。变量或泛型参数要与实际返回类型对应。

```kotlin
import top.sacz.xphelper.ext.*

val type = "Lcom/example/demo/DemoSettings;".toClass()
val constructor = "Lcom/example/demo/DemoSettings;-><init>()V".toConstructor()
val settings = constructor.newInstance()

val label = settings.callMethod<String>("rename", "Kotlin 示例")
settings.setFieldValue("title", "已修改")
val title = settings.getFieldValue<String>("title")
val version = type.callStaticMethod<String>("version")

val method = "Lcom/example/demo/DemoSettings;->rename(Ljava/lang/String;)Ljava/lang/String;"
    .toMethod()
val descriptor = method.descriptor
```

描述符是 DEX 里表示类和成员的写法。以 `Lcom/example/demo/DemoSettings;->rename(Ljava/lang/String;)Ljava/lang/String;` 为例，箭头左边是声明类，括号内是参数类型，括号后是返回类型。 `I` 表示 `int`，`V` 表示 `void`，`[I` 表示 `int[]`。字段描述符在字段名后用 `:` 接类型，例如 `title:Ljava/lang/String;`。

扩展函数默认使用宿主加载器。手头已有来自其他加载器的成员描述符时，可给 `toMethod(loader)`、`toField(loader)` 或 `toConstructor(loader)` 传明确的加载器。实例方法有多个重载且实参包含 `null` 时，优先使用反射查询的 `params(...)` 明确指定参数类型。

### 扩展 API 速查

| 接收者 | 用法 |
| --- | --- |
| `Method`、`Field`、`Constructor<*>` | `member.descriptor` 生成 DEX 描述符。 `method.toMethodFinder()` 和 `field.toFieldFinder()` 转为 DexKit 查询器。 |
| `String` | `"Lcom/example/Demo;".toClass()` 查类。 `"Lcom/example/Demo;->run()V".toMethod()`、`"Lcom/example/Demo;->title:Ljava/lang/String;".toField()`、`"Lcom/example/Demo;-><init>()V".toConstructor()` 解析成员。后三者均可传入 `clsLoader` 覆盖默认加载器。 |
| `Any` | `obj.callMethod<T>("run", args...)` 根据实参类型匹配并调用实例方法，`null` 实参作为类型通配。 `obj.getFieldValue<T>(name)`、`getFieldValue<T>(type)`、`getFieldValue<T>(name, type)` 读取字段。 `setFieldValue(name, value)`、`setFieldValue(type, value)`、`setFieldValue(name, type, value)` 写入字段。 |
| `Class<*>` | `clazz.callStaticMethod<T>(name, args...)` 调用静态方法。 `clazz.getStaticFieldValue<T>(name = "TAG")`、`clazz.setStaticFieldValue(name = "TAG", value = "Demo")` 读写静态字段，两者都支持 `type` 条件。 |

## 底层工具

### 缓存与描述符

下面这些接口适用于直接操作缓存、转换 DEX 描述符或维护 Activity 代理时。普通查询会自行读写缓存。在开发过程中想检查自定义查询结果，可以用对应的 `get…List(key)` 取回。手动写入时，使用与自定义查询相同的 `DexKitCache` 入口保持键的处理一致。

```kotlin
import top.sacz.xphelper.dexkit.cache.DexKitCache
import top.sacz.xphelper.dexkit.cache.DexKitCacheJson

val cached = DexKitCache.getMethodList("demo:displayName:v1")
val restored = cached?.firstOrNull()?.let { method ->
    val json = DexKitCacheJson.encodeMethod(method)
    DexKitCacheJson.decodeMethod(json)
}
```

#### `DexKitCache`

- `exist(key)` 判断键是否存在。
- `putClassList`、`putMethodList`、`putFieldList`、`putConstructorList` 用 `(key, list)` 写入反射结果。
- `getClassList`、`getMethodList`、`getFieldList`、`getConstructorList` 用 `(key)` 读取，缺失返回 `null`。
- `checkCacheExpired(context)` 在宿主版本变化时清空旧缓存。
- `clearCache()` 清空缓存，`getAllMethodString()` 用于以文本查看方法缓存。

#### `DexKitCacheProxy`

- 初始化配置目录后可用 `DexKitCacheProxy()` 直接操作缓存。
- 提供与 `DexKitCache` 同名的四组 `put…List` / `get…List`、`checkCacheExpired(context)` 和 `clearCache()`，按传入的原始键存取。
- `keys()` 返回键集合，`toString()` 按方法缓存格式输出，`findClassByJSONString(json)` 从类的缓存 JSON 还原 `Class`。

#### `DexKitCacheJson`

- `encodeClass`、`encodeMethod`、`encodeField`、`encodeConstructor` 各接收一个反射对象，返回 JSON。
- `decodeClass`、`decodeMethod`、`decodeField`、`decodeConstructor` 使用已配置的加载器从 JSON 还原。
- 公开数据类 `DexKitCacheJson.ConstructorInfo(declareClass, params)` 表示声明类名与参数类型名列表，支持解构、`copy()`、值比较、哈希、字符串表示及 `serializer()`。

#### `CacheKeyCompressor`

- `compressCacheKey(key)` 规范化缓存键，空值转为空字符串，超过 128 字符时使用首尾片段加哈希。

#### `DexMethodDescriptor`

- 构造器接收 `Method`、`Constructor`、完整描述符，或 `(classDescriptor / clazz, name, signature)`。
- `getMethodTypeSig(method)`、`getConstructorTypeSig(ctor)` 获取成员类型签名，`getTypeSig(clazz)` 获取类的类型描述符。
- `splitParameterTypes(s)` 拆分括号内拼接的参数描述符。
- `getParameterTypes()`、`getReturnType()` 从当前签名读取参数和返回类型描述符。
- `getDeclaringClassName()` 取点分类名，`getDescriptor()` 和 `toString()` 取完整描述符。
- `getMethodInstance(loader)`、`getConstructorInstance(loader)` 还原反射对象。
- `equals(other)` 和 `hashCode()` 按描述符比较和计算哈希。

#### `DexFieldDescriptor`

- 构造器接收 `Field`、完整描述符，或 `(classDescriptor, name, typeDescriptor)`。
- `getTypeSig(clazz)` 生成类型描述符，`getDeclaringClassName()` 取点分类名，`getDescriptor()` 和 `toString()` 取完整描述符。
- `getFieldInstance(loader)` 还原字段，`equals(other)` 和 `hashCode()` 按描述符比较和计算哈希。

#### `CheckClassType`

- `checkType(memberType, requestedType, matchParentClass)` 比较类型，支持源码定义的基本类型与装箱类型配对。
- 传 `true` 会额外检查双向可赋值性。

#### `AESHelper`

- `encrypt(content, password)` 与 `decrypt(content, password)` 均支持 `byte[]` 和 `String` 重载。
- 字符串使用 UTF-8 和十六进制密文，字节重载直接读写字节。
- 使用相同密码解密，失败返回 `null`。

#### `ReflectException`

- 可用 `()`、`(message)`、`(message, exception)` 构造。
- `hasOtherExceptions()` 检查是否携带原异常，`getOtherExceptions()` 获取原异常。

### Activity 代理回调

这些方法由 Android 框架在代理安装后调用。 `ProxyHandler(defaultCallback).handleMessage(msg)` 处理
Activity 启动消息并转交原回调。
`IActivityManagerHandler(activityManager).invoke(proxy, method, args)` 是活动管理器的调用处理器。
`ProxyInstrumentation.create(base)` 按系统版本创建包装器，构造器 `ProxyInstrumentation(base)` 接收待代理的
Instrumentation。它的公开方法沿用 Android Instrumentation 的参数。

| 用途 | `ProxyInstrumentation` 方法 |
| --- | --- |
| 创建与生命周期 | `newActivity`（两个重载，其中按类名创建的重载支持模块 Activity）、`newApplication`、`callApplicationOnCreate`、`callActivityOnCreate`（两个重载，转发前注入资源）、`callActivityOnDestroy`、`callActivityOnRestoreInstanceState`（两个重载）、`callActivityOnPostCreate`（两个重载）、`callActivityOnNewIntent`、`callActivityOnStart`、`callActivityOnRestart`、`callActivityOnPause`、`callActivityOnResume`、`callActivityOnStop`、`callActivityOnUserLeaving`、`callActivityOnSaveInstanceState`（两个重载）、`callActivityOnPictureInPictureRequested` |
| 运行与结果 | `onCreate`、`start`、`onStart`、`onException`、`onDestroy`、`sendStatus`、`addResults`、`finish`、`getContext`、`getComponentName`、`getTargetContext`、`getProcessName` |
| 主线程与监视 | `waitForIdle`、`waitForIdleSync`、`runOnMainSync`、`startActivitySync`（两个重载）、`addMonitor`（三个重载）、`checkMonitorHit`、`waitForMonitor`、`waitForMonitorWithTimeout`、`removeMonitor`、`acquireLooperManager`（`create(base)` 在 API 26 及以上返回的代理会转发） |
| 输入与自动化 | `setInTouchMode`、`invokeContextMenuAction`、`invokeMenuActionSync`、`sendCharacterSync`、`sendKeyDownUpSync`、`sendKeySync`、`sendPointerSync`、`sendStringSync`、`sendTrackballEventSync`、`getUiAutomation`（两个重载） |
| 性能统计 | `setAutomaticPerformanceSnapshots`、`startPerformanceSnapshot`、`endPerformanceSnapshot`、`isProfiling`、`startProfiling`、`stopProfiling`、`startAllocCounting`、`stopAllocCounting`、`getAllocCounts`、`getBinderCounts` |

## 常见问题

| 现象 | 排查方法 |
| --- | --- |
| 没有初始化日志 | 检查 `assets/xposed_init` 中的类名、模块是否启用、测试应用是否在作用域内，以及示例中包名和进程名是否与测试应用一致。修改设置后重启宿主进程。 |
| `findClass` 找不到类 | 确认已调用 `initContext`，并确认该类确实在宿主 APK 中。模块自身的类可使用 `ClassUtils.getModuleClassLoader()`。 |
| 反射找不到预期的重载 | 对照声明的参数顺序、类型和返回类型。先用 `getResult()` 看候选，再补齐条件。无参构造器可用 `<init>()V` 描述符精确定位。 |
| DexKit 返回空列表 | 先确认 Bridge 扫描的是预期 APK。检查包名限制、优化后是否仍保留常量和调用关系，以及当前缓存是否记录了空结果。调试时可放宽条件并清理 DexKit 缓存后重查。 |
| 创建配置时报存储路径为空 | 先初始化宿主 Context 或调用 `ConfigUtils.initialize(context / path)`，再创建 `ConfigUtils` 对象。 |
| 读不到模块应用里保存的配置 | 检查读写两端的目录、存储名和键。模块自己的 `filesDir` 与宿主应用的 `filesDir` 分别存放数据。 |
| 模块页面打不开或资源找不到 | 确认页面继承对应基类，从宿主 Activity 在主线程启动，且宿主已完成代理初始化。检查是否在使用资源之前调用了基类的 `onCreate` 或显式注入，并确认使用的是模块的 `R`。 |

## 参考

[项目 Wiki](https://github.com/suzhelan/XPHelper/wiki) · [EzXHelper](https://github.com/KyuubiRan/EzXHelper) · [DexKit](https://github.com/LuckyPray/DexKit) · [FastKV](https://github.com/BillyWei01/FastKV) · [提交问题](https://github.com/suzhelan/XPHelper/issues)
