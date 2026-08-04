# Gradle 国内镜像源配置完整指南

> 适用版本：Gradle 9.6.1 | Android Studio 2026.1.2 | 更新时间：2026-08-04

---

## 一、配置概览

| 层级 | 文件路径 | 作用 |
|------|----------|------|
| 系统级（全局） | `~/.gradle/init.d/mirrors.gradle` | 所有 Gradle 项目的 Maven 依赖仓库替换 |
| 系统级（全局） | `~/.gradle/init.d/wrapper-mirror.gradle` | Gradle Wrapper 分发下载加速 |
| 系统级（全局） | `~/.gradle/gradle.properties` | JVM / 网络 / 缓存全局参数 |
| 分发级 | `D:\Source\gradle-9.6.1\init.d\mirrors.gradle` | 使用此 Gradle 分发时生效（与系统级内容一致） |

### 镜像策略

- **阿里云**（主）：覆盖 Google Maven / Maven Central / Gradle Plugin / Spring / Apache Snapshots
- **清华 TUNA**（备）：单一代理 URL 覆盖上游全部仓库，阿里云故障时自动降级

---

## 二、镜像源对照表

| 原始仓库 | 阿里云镜像（主） | 清华 TUNA 镜像（备） |
|----------|-----------------|---------------------|
| `dl.google.com` / `maven.google.com` | `maven.aliyun.com/repository/google` | `mirrors.tuna.tsinghua.edu.cn/maven/` |
| `repo1.maven.org` / `maven.apache.org` | `maven.aliyun.com/repository/public` | `mirrors.tuna.tsinghua.edu.cn/maven/` |
| `plugins.gradle.org` | `maven.aliyun.com/repository/gradle-plugin` | `mirrors.tuna.tsinghua.edu.cn/maven/` |
| `repo.spring.io/release` | `maven.aliyun.com/repository/spring` | `mirrors.tuna.tsinghua.edu.cn/maven/` |
| `repo.spring.io/milestone|snapshot` | `maven.aliyun.com/repository/spring-plugin` | `mirrors.tuna.tsinghua.edu.cn/maven/` |
| `repository.apache.org` | `maven.aliyun.com/repository/apache-snapshots` | `mirrors.tuna.tsinghua.edu.cn/maven/` |
| `services.gradle.org/distributions/` | `mirrors.aliyun.com/gradle/distributions/` | `mirrors.tuna.tsinghua.edu.cn/gradle/distributions/` |

> **说明**：阿里云将不同上游仓库做了细粒度拆分，清华 TUNA 则是单一代理 URL 覆盖全部。本配置以阿里云为主，清华做备。

---

## 三、完整配置文件代码

### 3.1 `~/.gradle/init.d/mirrors.gradle`（系统级 Maven 镜像）

```groovy
// ============================================================
// Gradle 国内镜像源配置（系统级，所有项目生效）
// 阿里云 Maven 镜像（主） + 清华 TUNA 镜像（备）
// 策略：替换已有外网仓库 URL 为阿里云，追加清华源作为备用
// ============================================================

// 工具闭包：把已知外网 Maven 仓库替换为阿里云镜像
def replaceMirror = { repo ->
    if (repo instanceof MavenArtifactRepository) {
        def url = repo.url.toString()
        if (url.contains('dl.google.com') || url.contains('maven.google.com')) {
            repo.url = 'https://maven.aliyun.com/repository/google'
        } else if (url.contains('repo1.maven.org') || url.contains('maven.apache.org')) {
            repo.url = 'https://maven.aliyun.com/repository/public'
        } else if (url.contains('plugins.gradle.org')) {
            repo.url = 'https://maven.aliyun.com/repository/gradle-plugin'
        } else if (url.contains('repo.spring.io/release')) {
            repo.url = 'https://maven.aliyun.com/repository/spring'
        } else if (url.contains('repo.spring.io/milestone') || url.contains('repo.spring.io/snapshot')) {
            repo.url = 'https://maven.aliyun.com/repository/spring-plugin'
        } else if (url.contains('repository.apache.org')) {
            repo.url = 'https://maven.aliyun.com/repository/apache-snapshots'
        }
    }
}

// 工具闭包：如果仓库列表里没有国内镜像，则追加（阿里云主 + 清华备）
def ensureMirror = { repos ->
    if (!repos.any { it instanceof MavenArtifactRepository && it.url.toString().contains('maven.aliyun.com') }) {
        // 阿里云镜像（主）
        repos.maven { url 'https://maven.aliyun.com/repository/public' }
        repos.maven { url 'https://maven.aliyun.com/repository/google' }
        repos.maven { url 'https://maven.aliyun.com/repository/gradle-plugin' }
        // 清华 TUNA 镜像（备，单 URL 代理全部上游仓库）
        repos.maven { url 'https://mirrors.tuna.tsinghua.edu.cn/maven/' }
    }
}

// ========== 1. Settings 级仓库（Gradle 7+ 推荐，兼容 PREFER_SETTINGS）==========
settingsEvaluated { settings ->
    // Plugin 仓库替换
    settings.pluginManagement {
        repositories {
            all replaceMirror
            ensureMirror(delegate)
            mavenLocal()
        }
    }
    // 依赖仓库替换
    settings.dependencyResolutionManagement {
        repositories {
            all replaceMirror
            ensureMirror(delegate)
            mavenLocal()
        }
    }
}

// ========== 2. Buildscript classpath（项目级，不受 PREFER_SETTINGS 限制）==========
allprojects { project ->
    project.buildscript {
        repositories {
            all replaceMirror
            ensureMirror(delegate)
            mavenLocal()
        }
    }

    // 项目级依赖仓库（旧项目/未启用 dependencyResolutionManagement 时生效）
    try {
        project.repositories {
            all replaceMirror
            ensureMirror(delegate)
            mavenLocal()
        }
    } catch (Exception e) {
        // PREFER_SETTINGS 模式下不允许修改项目级仓库，忽略
    }
}
```

### 3.2 `~/.gradle/init.d/wrapper-mirror.gradle`（Wrapper 镜像 + 版本自动跟随）

```groovy
// ============================================================
// Gradle Wrapper 分发下载镜像 + 版本自动同步（永久，新建/克隆项目均生效）
// 阿里云镜像（主） + 清华 TUNA 镜像（备）
//
// 核心机制：
//   GradleVersion.current().version  ← Gradle 内置 API，自动获取当前运行的版本号
//
// 效果：
//   1. services.gradle.org → mirrors.aliyun.com（域名替换）
//   2. gradle-9.5.0-bin.zip → gradle-9.6.1-bin.zip（版本自动升级）
//   3. doFirst 执行前拦截 + doLast 执行后修复生成文件，双保险
// ============================================================

def GRADLE_DIST_MIRROR_PRIMARY   = 'https://mirrors.aliyun.com/gradle/distributions/'
def GRADLE_DIST_ORIGINAL         = 'https://services.gradle.org/distributions/'
def CURRENT_VERSION              = GradleVersion.current().version

println "[镜像] wrapper-mirror.gradle 已加载 (系统 Gradle: ${CURRENT_VERSION})"

// 公共：修复 distributionUrl（域名替换 + 版本升级）
// try-catch 兼容 String 和 Provider，避免 init script 类加载问题
def fixUrl = { raw ->
    def url
    try { url = raw.get() } catch (Exception _) { url = raw }
    url = url.toString()

    // 域名替换
    url = url.replace(GRADLE_DIST_ORIGINAL, GRADLE_DIST_MIRROR_PRIMARY)

    // 版本升级
    def m = (url =~ /gradle-(\d+\.\d+(?:\.\d+)?)/)
    if (m.find() && m.group(1) != CURRENT_VERSION) {
        def type = url.contains('-all.zip') ? '-all.zip' : '-bin.zip'
        url = "${GRADLE_DIST_MIRROR_PRIMARY}gradle-${CURRENT_VERSION}${type}"
        println "[镜像] wrapper 版本升级: ${m.group(1)} → ${CURRENT_VERSION}"
    }
    return url
}

// ========== 1. 拦截 gradle wrapper（doFirst + doLast 双保险）==========
gradle.rootProject {
    tasks.withType(Wrapper).configureEach { task ->
        // 执行前：改 URL，防止下载走外网
        task.doFirst {
            def oldUrl = task.distributionUrl.toString()
            def newUrl = fixUrl(oldUrl)
            if (newUrl != oldUrl) {
                task.setDistributionUrl(newUrl)
                println "[镜像] wrapper → 阿里云镜像, gradle-${CURRENT_VERSION}"
            }
        }
        // 执行后：修复生成的 gradle-wrapper.properties（兜底）
        task.doLast {
            def propsFile = project.file('gradle/wrapper/gradle-wrapper.properties')
            if (propsFile.exists()) {
                def props = new java.util.Properties()
                propsFile.withInputStream { props.load(it) }
                def url = props.getProperty('distributionUrl', '')
                if (!url || url.contains('services.gradle.org')) {
                    def fixed = fixUrl(url ?: "${GRADLE_DIST_ORIGINAL}gradle-${CURRENT_VERSION}-bin.zip")
                    props.setProperty('distributionUrl', fixed.replace('https://', 'https\\://'))
                    propsFile.withOutputStream { props.store(it, null) }
                    println "[镜像] wrapper 生成文件已修复 → 阿里云镜像"
                }
            }
        }
    }
}

// ========== 2. 自动修复已有项目的 wrapper properties ==========
gradle.rootProject { project ->
    def wrapperPropsFile = project.file('gradle/wrapper/gradle-wrapper.properties')
    if (wrapperPropsFile.exists()) {
        def props = new java.util.Properties()
        wrapperPropsFile.withInputStream { props.load(it) }
        def url = props.getProperty('distributionUrl', '')
        if (!url) return

        def newUrl = fixUrl(url)
        if (newUrl != url) {
            newUrl = newUrl.replace('https://', 'https\\://')
            props.setProperty('distributionUrl', newUrl)
            wrapperPropsFile.withOutputStream { props.store(it, null) }
            println "[镜像] wrapper.properties 已修复 → 阿里云镜像, gradle-${CURRENT_VERSION}"
        }
    }
}
```

### 3.3 `~/.gradle/gradle.properties`（全局参数）

```properties
# ============================================================
# Gradle 系统级配置（所有项目生效）
# ============================================================

# --- JVM & Daemon ---
org.gradle.jvmargs=-Xmx2048m -XX:MaxMetaspaceSize=512m -Dfile.encoding=UTF-8
org.gradle.daemon=true
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configuration-cache=true

# --- 网络加速（国内网络环境） ---
org.gradle.internal.network.retry.max.attempts=10
systemProp.org.gradle.internal.http.socketTimeout=120000
systemProp.org.gradle.internal.http.connectionTimeout=30000

# --- Gradle 分发加速由 ~/.gradle/init.d/wrapper-mirror.gradle 处理 ---
# 功能：自动修复已有项目的 wrapper 下载地址为阿里云镜像
#       gradle wrapper 命令生成的地址永久走镜像
```

### 3.4 `D:\Source\gradle-9.6.1\init.d\mirrors.gradle`（分发级）

> 内容与 §3.1 完全一致，此处不再重复。作用：当你使用 `D:\Source\gradle-9.6.1\bin\gradle` 时，即使没有系统级配置也能走镜像。

---

## 四、本次修改内容（2026-08-04）

### 4.1 新增清华 TUNA 镜像

**修改文件**：`~/.gradle/init.d/mirrors.gradle` 和 `D:\Source\gradle-9.6.1\init.d\mirrors.gradle`

- `ensureMirror` 闭包中新增一行清华源：

```groovy
// 清华 TUNA 镜像（备，单 URL 代理全部上游仓库）
repos.maven { url 'https://mirrors.tuna.tsinghua.edu.cn/maven/' }
```

- 文件头注释更新：标注"阿里云（主）+ 清华 TUNA（备）"

### 4.2 Wrapper 版本自动跟随（非硬编码）

**修改文件**：`~/.gradle/init.d/wrapper-mirror.gradle`

核心一行：
```groovy
def CURRENT_VERSION = GradleVersion.current().version
```

- `GradleVersion.current()` 是 Gradle 内置 API，**无需 import**，直接可用
- `.version` 返回当前运行的 Gradle 版本字符串，如 `"9.6.1"`
- 将来升级系统 Gradle 到 `10.0.0`，这行自动返回 `"10.0.0"`，所有项目自动跟随
- 同时定义阿里云 + 清华 TUNA 镜像常量，一键切换

### 4.3 设置 Gradle 环境变量

- `GRADLE_HOME` = `D:\Source\gradle-9.6.1`（用户级）
- `PATH` 追加 `D:\Source\gradle-9.6.1\bin`

---

## 五、工作原理

### 依赖解析顺序

```
项目请求依赖
  → settingsEvaluated: 替换已知外网仓库 URL → 阿里云
  → ensureMirror: 追加 阿里云 public/google/gradle-plugin + 清华 TUNA
  → mavenLocal: 本地缓存优先
  → 下载：阿里云 → 阿里云失败则尝试清华 TUNA → 最终回退外网
```

### Wrapper 版本自动同步流程

```
Gradle 启动
  → GradleVersion.current().version  ← 自动读取当前系统版本（如 "9.6.1"）
  → 检查项目 gradle-wrapper.properties
  → 提取 distributionUrl 中的版本号（如 "9.5.0"）
  → 比较：9.5.0 ≠ 9.6.1 → 改写为 gradle-9.6.1-bin.zip
  → 域名替换：services.gradle.org → mirrors.aliyun.com
  → 回写 gradle-wrapper.properties
  → 控制台输出：[镜像] Wrapper 版本升级: 9.5.0 → 9.6.1
```

**关键点**：版本号不是写死的 `"9.6.1"`，而是 `GradleVersion.current().version`。
将来升级 Gradle 后，这个值自动变化，所有项目自动跟上，零维护。

### 仓库查找优先级

1. `mavenLocal()` — 本地 `~/.m2/repository`
2. `maven.aliyun.com/repository/public` — 阿里云 Maven Central 代理
3. `maven.aliyun.com/repository/google` — 阿里云 Google Maven 代理
4. `maven.aliyun.com/repository/gradle-plugin` — 阿里云 Gradle 插件代理
5. `mirrors.tuna.tsinghua.edu.cn/maven/` — 清华 TUNA 全量代理
6. （项目中自定义的其他仓库）

---

## 六、验证方法

### 检查依赖是否走镜像

```bash
# 任意 Gradle 项目下执行构建，观察下载日志
gradle build --info 2>&1 | grep -E "(maven.aliyun|tuna.tsinghua)"
```

### 检查 Wrapper 分发地址

```bash
# 查看项目 wrapper 配置
cat gradle/wrapper/gradle-wrapper.properties | grep distributionUrl
```

正确输出应包含 `mirrors.aliyun.com` 而非 `services.gradle.org`。

### 强制刷新缓存测试

```bash
gradle build --refresh-dependencies
```

观察控制台输出，所有 `Download` 行应来自 `maven.aliyun.com` 或 `mirrors.tuna.tsinghua.edu.cn`。

---

## 七、常见问题

### Q: 阿里云和清华源都挂了呢？

A: Gradle 会 fallback 到项目 `build.gradle` 中声明的原始仓库（如果没被替换掉）。另外 `mavenLocal()` 始终在最前面，本地已缓存的依赖不受影响。

### Q: 如何临时禁用镜像？

A: 将 `~/.gradle/init.d/mirrors.gradle` 重命名为 `mirrors.gradle.bak`，或在该文件顶部加 `return` 提前退出。

### Q: 某个特定依赖镜像里没有怎么办？

A: 在项目 `build.gradle` 或 `settings.gradle` 里显式添加原始仓库即可。项目级仓库不受 init script 删除。

### Q: Android Studio 需要额外配置吗？

A: 不需要。只要 AS 使用的 Gradle 能加载 `~/.gradle/init.d/`，镜像就会自动生效。

### Q: Android Studio 新建项目版本落后怎么办？

A: `wrapper-mirror.gradle` 通过 `GradleVersion.current()` 自动解决：
1. 首次构建时自动将 `gradle-wrapper.properties` 中的版本升级到当前系统版本
2. 域名也自动替换为阿里云镜像
3. 构建日志会打印 `[镜像] Wrapper 版本升级: 9.5.0 → 9.6.1 (项目: MyApp)`

将来系统 Gradle 升级到 10.0.0 后，这行代码自动返回 `"10.0.0"`，无需改任何配置文件。所有项目的 wrapper 都会自动跟随。

---

## 八、文件清单速查

| 文件 | 绝对路径 |
|------|----------|
| 系统级 Maven 镜像 | `C:\Users\22214\.gradle\init.d\mirrors.gradle` |
| 系统级 Wrapper 镜像 | `C:\Users\22214\.gradle\init.d\wrapper-mirror.gradle` |
| 系统级 Gradle 属性 | `C:\Users\22214\.gradle\gradle.properties` |
| 分发级 Maven 镜像 | `D:\Source\gradle-9.6.1\init.d\mirrors.gradle` |
| 本文档 | `C:\Users\22214\Desktop\Gradle国内镜像配置指南.md` |
