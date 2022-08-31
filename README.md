# ComponentBus
ComponentBus 用于组件（module）间的通信, 而不必相互依赖.

## project build.gradle 内添加 ksp 插件
```groovy
plugins {
    id "com.google.devtools.ksp" version "1.7.10-1.0.6" apply false
}
```

## app module build.gradle 内添加插件、要扫描的jar包、ksp 生成文件引入
```groovy
plugins {
    id "com.google.devtools.ksp"
    id 'love.nuoyan.android.component_bus_register' version '0.0.1'
}

componentBusExt {
    jarNameScanList = [
            "classes", // jar包名称，构建时会打印所有jar包名称，把需要扫描的进行添加
            "xxx"
    ]
}

// app module 中添加如下代码
android {
    applicationVariants.all { variant ->
        kotlin.sourceSets {
            def name = variant.name
            getByName(name) {
                kotlin.srcDir("$buildDir/generated/ksp/$name/kotlin")
            }
        }
    }
}

// library module 中添加如下代码
android {
    libraryVariants.all { variant ->
        kotlin.sourceSets {
            def name = variant.name
            getByName(name) {
                kotlin.srcDir("$buildDir/generated/ksp/$name/kotlin")
            }
        }
    }
}
```

## module 内添加依赖
```groovy
    implementation "org.jetbrains.kotlin:kotlin-stdlib:1.7.10"
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-core:1.6.4'       // kotlin 协程
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.6.4'

    implementation 'love.nuoyan.component_bus:bus:0.0.1'
    ksp 'love.nuoyan.component_bus:processor:0.0.1'
```

## module 内添加组件 API
```kotlin
@Component(componentName = "组件名称")
object MainComponent {

    @Action(actionName = "事件名称")
    fun event1(params1: Int, params2: Int): Boolean {
        return params1 > params2
    }

    // 挂起函数，必须使用 call 函数调用
    @Action(actionName = "事件名称2", interceptorName = ["拦截器类名称"])
    suspend fun event2(params1: String, params2: String): String {
        return params1 + params2
    }

    // 可以直接返回对应的 Result
    @Action(actionName = "事件名称3")
    fun event3(params1: Int, params2: Int): Result<Boolean> {
        return Result.resultError(code = 1, msg = "参数 params1 应在 0-10 之间")
    }
}
```

## 添加拦截器
```kotlin
object TestInterceptor : IInterceptor {
    override suspend fun <T> intercept(chain: Chain): Result<T> {
        Log.e("TestInterceptor", "${chain.request.componentName} 执行了 TestInterceptor")
        return chain.proceed()
    }

    override fun <T> interceptSync(chain: Chain): Result<T> {
        Log.e("TestInterceptor", "${chain.request.componentName} 执行了 TestInterceptor Sync")
        return chain.proceedSync()
    }
}
```

## 添加全局拦截器
```kotlin
object LogGlobalInterceptor : GlobalInterceptor() {
    init {
        priority = 8    // 用于设置拦截器的优先级，越高执行顺序越靠前
    }
    
    override suspend fun <T> intercept(chain: Chain): Result<T> {
        Log.e("LogGlobalInterceptor", "${chain.request.componentName} 执行了 LogGlobalInterceptor 0")
        return chain.proceed()
    }

    override fun <T> interceptSync(chain: Chain): Result<T> {
        Log.e("LogGlobalInterceptor", "${chain.request.componentName} 执行了 LogGlobalInterceptor 0 Sync")
        return chain.proceedSync()
    }
}
```

## 调用组件 Action
```kotlin
val result = ComponentBus.with("组件名称", "事件名称")
    .params("params1", 1)
    .interceptors("TestInterceptor")
    .apply {
        params["params2"] = 2
        interceptors.add("TestInterceptor2") 
    }
    .callSync<Boolean>()
```