# XPHelper

[![JitPack](https://jitpack.io/v/suzhelan/XpHelper.svg)](https://jitpack.io/#suzhelan/XpHelper)

用于开发 Android Xposed 模块，提供反射工具、DexKit 查询与缓存、模块 Activity 与资源加载，以及基于 FastKV
的配置存储。

[接入](#接入) · [反射](#反射) · [DexKit](#dexkit) · [配置存储](#配置存储) · [Activity 与资源](#activity-与资源) · [Kotlin 扩展](#kotlin-扩展) · [底层工具](#底层工具)

## 接入

本仓库的 `:xphelper` 模块配置为 minSdk 24，构建使用 JDK 17。使用发布版时，以所选版本的构建配置为准。

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

| `XpHelper` 方法                     | 调用时机与用途                                                                                                                 |
|-------------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| `initZygote(startupParam)`          | 在 Xposed 的同名回调中保存模块 APK 路径。                                                                                      |
| `initContext(application)`          | 在目标应用进程拿到宿主的 Application Context 后调用。设置宿主类加载器、配置目录、Activity 代理，并按宿主版本检查 DexKit 缓存。 |
| `setConfigPath(pathDir)`            | 为后续创建的配置实例指定可写目录。若使用 `initContext`，在其后设置目录，因为它会重设默认目录。                                 |
| `setConfigPassword(password)`       | 为后续创建的配置实例设置全局密码及加密开关。                                                                                   |
| `injectResourcesToContext(context)` | 在 `initZygote` 后，把模块 APK 的资源加载到指定 Context。                                                                      |

`context`、`classLoader` 和 `moduleApkPath` 分别保存宿主上下文、宿主原始类加载器和模块 APK
路径。完整入口见 [InjectHook.java](app/src/main/java/top/sacz/hook/InjectHook.java)。

## 反射

以下示例假设已初始化，且 `obj` 所属类有 `rename(String)` 方法、`title` 字段和无参构造器。

```java
MethodUtils.create(obj).

methodName("rename").

params(String .class)
        .

invokeNoReturn(obj, "XPHelper");

String title = FieldUtils.create(obj).fieldName("title").firstValue(obj);
Object value = ConstructorUtils.create(obj.getClass()).paramTypes().newFirstInstance();
```

| 类                 | 公开方法及用途                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
|--------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `ClassUtils`       | `intiClassLoader(loader)` 单独初始化反射类加载器。 `getClassLoader()` 取支持缓存和类型描述符的宿主加载器，`getModuleClassLoader()` 取模块加载器。 `findClass(name)` 支持类名、DEX 类描述符及基本类型，失败抛异常。 `findClassOrNull(name)` 在查找失败时返回 `null`。 `isCommonlyUsedClass(name)` 检查是否属于内置的常见包名前缀。                                                                                                                                                                                                                                                      |
| `MethodUtils`      | `create(obj / clazz / className)` 创建查找器。 `methodName(name)`、`returnType(type)`、`params(types...)`、`paramCount(count)` 分别筛选名称、返回类型、参数类型与个数。 `invokeFirst(obj, args...)`、`invokeLast(obj, args...)` 调用首项或末项，`invokeNoReturn(obj, args...)` 忽略首项的返回值，`invokeFirstStatic(args...)` 调用首个匹配项作为静态方法。旧方法 `callFirst`、`callLast`、`callFirstStatic` 分别对应上述同名 `invoke` 方法。 `getDescriptor(method)` 生成 DEX 描述符，`getMethodByDescriptor(desc)` 或 `getMethodByDescriptor(desc, loader)` 解析为可访问的 `Method`。 |
| `FieldUtils`       | `create(obj / clazz / className)` 创建查找器。 `fieldName(name)` 和 `fieldType(type)` 筛选名称和类型。 `firstValue(obj)`、`lastValue(obj)` 读取首项或末项，`setFirst(obj, value)`、`setLast(obj, value)` 写入首项或末项。 `getField(obj, name, type)` 按名称和类型读取，`setField(obj, name, value)` 按名称写入，`getFirstType(obj, type)` 按类型读取。 `setFirstType(obj, value)` 当前以值的类名筛选字段名，按类型写入可用 `create(obj).fieldType(type).setFirst(obj, value)`。 `getDescriptor(field)` 和 `getFieldByDescriptor(desc[, loader])` 在字段和描述符之间转换。             |
| `ConstructorUtils` | `create(obj / clazz / className)` 创建查找器。 `paramCount(count)` 筛选非零参数个数，使用 `paramTypes()` 指定无参构造器。 `paramTypes(types...)` 筛选参数类型，`newFirstInstance(args...)` 调用首个匹配的构造器。 `newInstance(clazz, types, args...)` 显式指定类型，`newInstance(clazz, args...)` 根据非空实参推断类型。                                                                                                                                                                                                                                                              |
| 三个反射查找器共有 | `matchParentClass(true)` 启用类型的双向可赋值匹配。参数位置可传 `Ignore.class` 作为通配，字段类型也可传 `Ignore.class`。调用静态方法或读写静态字段时，选择对应成员并传 `null` 作为实例。                                                                                                                                                                                                                                                                                                                                                                                               |
| `BaseFinder`       | `find()` 扫描当前类的声明成员。 `findAndSuper()` 在无结果时向父类查找并将结果设为可访问。 `getResult()` 返回列表，`get(index)` 按从零开始的下标取项，`first()`、`last()` 取首尾，无结果时抛 `ReflectException`，`firstOrNull()`、`lastOrNull()` 无结果时返回 `null`。取结果的方法会自动触发查找。 `buildSign()` 返回缓存签名。直接调用 `ConstructorUtils.find()` 时，缓存未命中会返回 `null`，取结果请使用 `getResult()` 等方法。                                                                                                                                                      |

## DexKit

初始化后可直接查询宿主 APK。下面按声明类和方法名查找，查不到时返回 `null`。

```kotlin
val method = MethodFinder.build()
    .declaredClass(demoClass)
    .methodName("rename")
    .paramCount(1)
    .firstOrNull()

val field = DexFinder.findField {
    declaredClass = demoClass
    fieldName = "title"
}.firstOrNull()
```

| API              | 公开方法及用途                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
|------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `DexFinder`      | `create(apkPath)` 在 Bridge 未创建时打开指定 APK。 `getDexKitBridge()` 获取 Bridge，首次按需打开宿主 APK，并重置自动释放计时。 `setAutoCloseTime(ms)` 设置计时，默认 10 秒，传 `0` 后续获取时不再安排自动关闭。 `close()` 释放 Bridge 并取消计时。 `clearCache()` 清空持久查询缓存。 `isLoadLibrary` 是库加载时使用的原子标记。                                                                                                                                                                                                                                                                                 |
| Kotlin DSL       | `DexFinder.findMethod { … }`、`findField { … }`、`findClass { … }` 分别配置 `MethodInfo`、`FieldInfo`、`ClassInfo` 并返回对应 Finder。独立创建这些配置对象时，可设置与 Finder 条件对应的属性，再调用 `generate()`。类的接口条件在 DSL 中使用 `interfaces`，修饰符使用 `modifiers` 和 `matchType`。                                                                                                                                                                                                                                                                                                              |
| 自定义缓存查询   | `DexFinder.queryMethod(key, block)`、`queryField(key, block)`、`queryClass(key, block)`、`queryConstructor(key, block)` 仅在缓存未命中时把 Bridge 传给回调。回调返回对应的反射对象列表。为不同结果类型与不同查询条件使用不同的键。                                                                                                                                                                                                                                                                                                                                                                              |
| 三个 Finder 共有 | `build()` 创建空查询，`from(method / field / clazz)` 用已有反射对象填充条件，其中 `ClassFinder.from(clazz)` 需要 `clazz` 有父类。 `modifiers(flags, matchType)` 按修饰符匹配，例如 `Modifier.PUBLIC` 配合 `MatchType.Contains`。 `searchPackages(packages...)` 和 `excludePackages(packages...)` 限定或排除包。 `find()` 返回全部结果，`first()` 无结果时抛异常，`firstOrNull()` 无结果返回 `null`。 `existCache()` 检查查询键是否存在，`toString()` 返回作为缓存键的查询描述。                                                                                                                                 |
| `MethodFinder`   | `declaredClass(clazz)`、`methodName(name)`、`returnType(type)`、`parameters(types...)`、`paramCount(count)` 匹配方法签名。零参用 `paramCount(0)`。 `usedString(strings...)`、`usingNumbers(numbers...)` 匹配使用的常量。 `usedFields(fields...)` 接收 `Field` 或 `FieldFinder`。 `invokeMethods(methods...)` 匹配被调用的方法，`callMethods(methods...)` 匹配调用方。 `findConstructor()` 返回构造器列表，`firstConstructor()` 和 `firstConstructorOrNull()` 分别在无结果时抛异常或返回 `null`。 `buildMethodMatcher()` 将当前条件转换为 DexKit Matcher，`toMethodMatcher(method)` 直接从反射对象构造 Matcher。 |
| `FieldFinder`    | `declaredClass(clazz)`、`fieldName(name)`、`fieldType(type)` 匹配字段。 `readMethods(methods...)` 和 `writeMethods(methods...)` 匹配读写该字段的方法，接收 `Method` 或 `MethodFinder`。 `buildFieldMatcher()` 转换当前条件，`toFieldMatcher(field)` 直接从反射对象构造 Matcher。                                                                                                                                                                                                                                                                                                                                |
| `ClassFinder`    | `className(name)`、`superClass(name)`、`addInterface(names...)` 匹配类名、父类与接口。 `usedString(strings...)` 匹配类中使用的字符串。 `fields(fieldFinders...)` 和 `methods(methodFinders...)` 匹配类中声明的成员。 `buildClassMatcher()` 将类条件转换为 DexKit Matcher。                                                                                                                                                                                                                                                                                                                                      |

Java 调用 Kotlin 对象的普通方法时使用 `DexFinder.INSTANCE`，`getDexKitBridge()` 和四个 `query*`
方法可直接静态调用。直接使用 Bridge 或 Matcher 时，在模块中声明与库版本匹配的 DexKit 依赖。

## 配置存储

可单独调用 `ConfigUtils.initialize(context)` 或 `initialize(path)` 后使用。Context 版本将目录设为
`filesDir/XpHelper`。`ConfigUtils()` 使用名为 `default` 的存储，`ConfigUtils(key)` 指定名称，
`ConfigUtils(key, password)` 指定实例密码。先调用 `setGlobalPassword(password)` 或
`XpHelper.setConfigPassword(password)` 设置非空全局密码来启用加密。全局密码为空时，传入实例密码仍会创建未加密的存储。FastKV
加密适配器会原样处理整数和长整型。

```kotlin
val config = ConfigUtils("settings")
config.put("enabled", true)
val enabled = config.getBoolean("enabled", false)
config.putStringList("tags", listOf("demo"))
val tags = config.getStringList("tags")
```

| 方法                                                                                | 用途                                                                                                                                                                                                   |
|-------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `put(key, value)`                                                                   | 写入字符串、基本数值、布尔值、字节数组，或将列表、Map 与可序列化对象写为 JSON。                                                                                                                        |
| `getInt`、`getDouble`、`getLong`、`getBoolean`、`getFloat`、`getBytes`、`getString` | 按 `(key, def)` 读取对应类型。Kotlin 可省略 `def`，默认依次为 `0`、`0.0`、`0L`、`false`、`0f`、空字节数组、空字符串。Java 须传默认值。                                                                 |
| `putStringList(key, list)`、`getStringList(key)`                                    | 读写字符串列表，缺失时返回空列表。                                                                                                                                                                     |
| `putObject(key, value)`、`getObject<T>(key)`                                        | 使用 kotlinx.serialization 读写对象，缺失时返回 `null`。自定义类型可用 `@Serializable` 声明。重载可在末尾传 `KSerializer<T>` 或 `Class<T>`，Java 可使用 `config.getObject("profile", Profile.class)`。 |
| `putList(key, mutableList)`、`getList<T>(key)`                                      | 读写对象列表，缺失时返回空的可变列表。重载可在末尾传元素的 `KSerializer<T>` 或 `Class<T>`。                                                                                                            |
| `putAll(map)`、`toMap()`                                                            | 调用 FastKV 批量写入，或取出底层存储值。以 JSON 保存的对象在 Map 中为字符串。                                                                                                                          |
| `getAllKeys()`、`containsKey(key)`、`remove(key)`、`clearAll()`                     | 列出键、检查键、删除单项和清空当前存储。                                                                                                                                                               |

## Activity 与资源

继承 `BaseActivity`（View）或 `BaseComposeActivity`（Compose），并在覆写的 `onCreate(savedInstanceState)`
中先调用 `super.onCreate(savedInstanceState)` 加载模块资源。初始化后，从宿主 Activity 启动模块页面。

```kotlin
hostActivity.startActivity(Intent(hostActivity, ModuleActivity::class.java))
```

| API                                   | 公开方法及用途                                                                                                                                                                                                                                                                                                                                                                                             |
|---------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `ActivityProxyManager`                | `initActivityProxyManager(hostContext)` 初始化代理，由 `XpHelper.initContext` 自动调用。默认使用宿主清单中的首个 Activity，宿主需至少声明一个 Activity。 `isModuleActivity(className)` 检查类是否继承两种基类之一。 `HostActivityClassName` 保存默认代理类名，`ACTIVITY_PROXY_INTENT` 保存代理使用的 Intent 键。                                                                                           |
| `BaseActivity`、`BaseComposeActivity` | `getClassLoader()` 返回页面使用的组合加载器。基类的 `onRestoreInstanceState(state)` 会为视图状态设置加载器，再交还系统恢复。 `BaseActivity` 子类可调用受保护的 `requestTranslucentStatusBar()` 设置透明系统栏。 `BaseComposeActivity.BaseActivityClassLoader(moduleLoader)` 创建组合加载器，其 `loadClass(name, resolve)` 按包选择系统、宿主或模块加载器。                                                 |
| `ActivityTools`                       | `getAllActivity(context)` 读取该应用清单中的 Activity 信息。 `getAllChildViews(activity / rootView)` 递归列出子 View。 `getTopActivity()` 查找首个未暂停的 Activity，找不到时返回 `null`。 `runOnUiThread(task)` 在主线程执行任务，`runOnUiThreadDelay(ms, task)` 延迟执行。 `injectResourcesToContext(context, apkPath)` 加载指定 APK 的资源，Android 11 及以上从后台线程调用时会把资源挂载提交到主线程。 |

启动及资源打包见 [ActivityTest.kt](app/src/main/java/top/sacz/hook/ActivityTest.kt)
和 [app/build.gradle.kts](app/build.gradle.kts)。

## Kotlin 扩展

导入 `top.sacz.xphelper.ext.*`。

| 接收者                              | 用法                                                                                                                                                                                                                                                                                                      |
|-------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `Method`、`Field`、`Constructor<*>` | `member.descriptor` 生成 DEX 描述符。 `method.toMethodFinder()` 和 `field.toFieldFinder()` 转为 DexKit 查询器。                                                                                                                                                                                           |
| `String`                            | `"Lcom/example/Demo;".toClass()` 查类。 `"Lcom/example/Demo;->run()V".toMethod()`、`"Lcom/example/Demo;->title:Ljava/lang/String;".toField()`、`"Lcom/example/Demo;-><init>()V".toConstructor()` 解析成员。后三者均可传入 `clsLoader` 覆盖默认加载器。                                                    |
| `Any`                               | `obj.callMethod<T>("run", args...)` 根据实参类型匹配并调用实例方法，`null` 实参作为类型通配。 `obj.getFieldValue<T>(name)`、`getFieldValue<T>(type)`、`getFieldValue<T>(name, type)` 读取字段。 `setFieldValue(name, value)`、`setFieldValue(type, value)`、`setFieldValue(name, type, value)` 写入字段。 |
| `Class<*>`                          | `clazz.callStaticMethod<T>(name, args...)` 调用静态方法。 `clazz.getStaticFieldValue<T>(name = "TAG")`、`clazz.setStaticFieldValue(name = "TAG", value = "Demo")` 读写静态字段，两者都支持 `type` 条件。                                                                                                  |

## 底层工具

| API                   | 公开方法及用途                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
|-----------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `DexKitCache`         | `exist(key)` 判断键是否存在。 `putClassList`、`putMethodList`、`putFieldList`、`putConstructorList` 用 `(key, list)` 写入反射结果。 `getClassList`、`getMethodList`、`getFieldList`、`getConstructorList` 用 `(key)` 读取，缺失返回 `null`。 `checkCacheExpired(context)` 在宿主版本变化时清空旧缓存。 `clearCache()` 清空缓存，`getAllMethodString()` 用于以文本查看方法缓存。                                                                                                                                                                                                         |
| `DexKitCacheProxy`    | 初始化配置目录后可用 `DexKitCacheProxy()` 直接操作缓存。提供与 `DexKitCache` 同名的四组 `put…List` / `get…List`、`checkCacheExpired(context)` 和 `clearCache()`，按传入的原始键存取。 `keys()` 返回键集合，`toString()` 按方法缓存格式输出，`findClassByJSONString(json)` 从类的缓存 JSON 还原 `Class`。                                                                                                                                                                                                                                                                                |
| `DexKitCacheJson`     | `encodeClass`、`encodeMethod`、`encodeField`、`encodeConstructor` 各接收一个反射对象，返回 JSON。 `decodeClass`、`decodeMethod`、`decodeField`、`decodeConstructor` 使用已配置的加载器从 JSON 还原。公开数据类 `DexKitCacheJson.ConstructorInfo(declareClass, params)` 表示声明类名与参数类型名列表，支持解构、`copy()`、值比较、哈希、字符串表示及 `serializer()`。                                                                                                                                                                                                                    |
| `CacheKeyCompressor`  | `compressCacheKey(key)` 规范化缓存键，空值转为空字符串，超过 128 字符时使用首尾片段加哈希。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| `DexMethodDescriptor` | 构造器接收 `Method`、`Constructor`、完整描述符，或 `(classDescriptor / clazz, name, signature)`。 `getMethodTypeSig(method)`、`getConstructorTypeSig(ctor)` 获取成员类型签名，`getTypeSig(clazz)` 获取类的类型描述符。 `splitParameterTypes(s)` 拆分括号内拼接的参数描述符。 `getParameterTypes()`、`getReturnType()` 从当前签名读取参数和返回类型描述符。 `getDeclaringClassName()` 取点分类名，`getDescriptor()` 和 `toString()` 取完整描述符。 `getMethodInstance(loader)`、`getConstructorInstance(loader)` 还原反射对象。 `equals(other)` 和 `hashCode()` 按描述符比较和计算哈希。 |
| `DexFieldDescriptor`  | 构造器接收 `Field`、完整描述符，或 `(classDescriptor, name, typeDescriptor)`。 `getTypeSig(clazz)` 生成类型描述符，`getDeclaringClassName()` 取点分类名，`getDescriptor()` 和 `toString()` 取完整描述符。 `getFieldInstance(loader)` 还原字段，`equals(other)` 和 `hashCode()` 按描述符比较和计算哈希。                                                                                                                                                                                                                                                                                 |
| `CheckClassType`      | `checkType(memberType, requestedType, matchParentClass)` 比较类型，支持源码定义的基本类型与装箱类型配对。传 `true` 会额外检查双向可赋值性。                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| `AESHelper`           | `encrypt(content, password)` 与 `decrypt(content, password)` 均支持 `byte[]` 和 `String` 重载。字符串使用 UTF-8 和十六进制密文，字节重载直接读写字节。使用相同密码解密，失败返回 `null`。                                                                                                                                                                                                                                                                                                                                                                                               |
| `ReflectException`    | 可用 `()`、`(message)`、`(message, exception)` 构造。 `hasOtherExceptions()` 检查是否携带原异常，`getOtherExceptions()` 获取原异常。                                                                                                                                                                                                                                                                                                                                                                                                                                                    |

### Activity 代理回调

这些方法由 Android 框架在代理安装后调用。 `ProxyHandler(defaultCallback).handleMessage(msg)` 处理
Activity 启动消息并转交原回调。
`IActivityManagerHandler(activityManager).invoke(proxy, method, args)` 是活动管理器的调用处理器。
`ProxyInstrumentation.create(base)` 按系统版本创建包装器，构造器 `ProxyInstrumentation(base)` 接收待代理的
Instrumentation。它的公开方法沿用 Android Instrumentation 的参数。

| 用途           | `ProxyInstrumentation` 方法                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
|----------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 创建与生命周期 | `newActivity`（两个重载，其中按类名创建的重载支持模块 Activity）、`newApplication`、`callApplicationOnCreate`、`callActivityOnCreate`（两个重载，转发前注入资源）、`callActivityOnDestroy`、`callActivityOnRestoreInstanceState`（两个重载）、`callActivityOnPostCreate`（两个重载）、`callActivityOnNewIntent`、`callActivityOnStart`、`callActivityOnRestart`、`callActivityOnPause`、`callActivityOnResume`、`callActivityOnStop`、`callActivityOnUserLeaving`、`callActivityOnSaveInstanceState`（两个重载）、`callActivityOnPictureInPictureRequested` |
| 运行与结果     | `onCreate`、`start`、`onStart`、`onException`、`onDestroy`、`sendStatus`、`addResults`、`finish`、`getContext`、`getComponentName`、`getTargetContext`、`getProcessName`                                                                                                                                                                                                                                                                                                                                                                                    |
| 主线程与监视   | `waitForIdle`、`waitForIdleSync`、`runOnMainSync`、`startActivitySync`（两个重载）、`addMonitor`（三个重载）、`checkMonitorHit`、`waitForMonitor`、`waitForMonitorWithTimeout`、`removeMonitor`、`acquireLooperManager`（`create(base)` 在 API 26 及以上返回的代理会转发）                                                                                                                                                                                                                                                                                  |
| 输入与自动化   | `setInTouchMode`、`invokeContextMenuAction`、`invokeMenuActionSync`、`sendCharacterSync`、`sendKeyDownUpSync`、`sendKeySync`、`sendPointerSync`、`sendStringSync`、`sendTrackballEventSync`、`getUiAutomation`（两个重载）                                                                                                                                                                                                                                                                                                                                  |
| 性能统计       | `setAutomaticPerformanceSnapshots`、`startPerformanceSnapshot`、`endPerformanceSnapshot`、`isProfiling`、`startProfiling`、`stopProfiling`、`startAllocCounting`、`stopAllocCounting`、`getAllocCounts`、`getBinderCounts`                                                                                                                                                                                                                                                                                                                                  |

## 参考

[项目 Wiki](https://github.com/suzhelan/XPHelper/wiki) · [EzXHelper](https://github.com/KyuubiRan/EzXHelper) · [DexKit](https://github.com/LuckyPray/DexKit) · [FastKV](https://github.com/BillyWei01/FastKV) · [提交问题](https://github.com/suzhelan/XPHelper/issues)
