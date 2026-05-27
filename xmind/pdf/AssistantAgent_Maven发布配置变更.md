# AssistantAgent Maven 发布配置变更总结

## 变更日期
2026-04-07

## 变更目标
将 AssistantAgent 项目的 Jar 包发布仓库从阿里内部仓库迁移到顺丰 Maven 仓库。

---

## 一、修改内容

### 1. 根 pom.xml 修改

**文件路径**: `/Users/01428674/Ai/sds-spring-ai-project/AssistantAgent/pom.xml`

| 配置项 | 修改前 | 修改后 |
|--------|--------|--------|
| groupId | `com.alibaba.agent.assistant` | `com.sftech.agent.assistant` |
| version | `0.2.4` | `0.2.4-SNAPSHOT` |
| snapshotRepository | `http://repo.alibaba-inc.com/mvn/snapshots` | `http://repo.sftcwl.com/repository/sf-snapshot` |
| releaseRepository | `http://repo.alibaba-inc.com/mvn/releases` | 已删除 |

**修改详情**:
```xml
<!-- 修改前 -->
<groupId>com.alibaba.agent.assistant</groupId>
<artifactId>assistant-agent</artifactId>
<version>0.2.4</version>
<distributionManagement>
    <repository>
        <id>releases</id>
        <url>http://repo.alibaba-inc.com/mvn/releases</url>
    </repository>
    <snapshotRepository>
        <id>snapshots</id>
        <url>http://repo.alibaba-inc.com/mvn/snapshots</url>
    </snapshotRepository>
</distributionManagement>

<!-- 修改后 -->
<groupId>com.sftech.agent.assistant</groupId>
<artifactId>assistant-agent</artifactId>
<version>0.2.4-SNAPSHOT</version>
<distributionManagement>
    <snapshotRepository>
        <id>sf-snapshot</id>
        <url>http://repo.sftcwl.com/repository/sf-snapshot</url>
    </snapshotRepository>
</distributionManagement>
```

同时修改了 `<dependencyManagement>` 中所有内部依赖的 groupId:
- `com.alibaba.agent.assistant` → `com.sftech.agent.assistant`

---

### 2. 子模块 pom.xml 修改

**涉及模块** (7个):
- `assistant-agent-common`
- `assistant-agent-core`
- `assistant-agent-extensions`
- `assistant-agent-prompt-builder`
- `assistant-agent-evaluation`
- `assistant-agent-autoconfigure`
- `assistant-agent-start`

**每个子模块修改内容**:
```xml
<!-- 修改前 -->
<parent>
    <groupId>com.alibaba.agent.assistant</groupId>
    <artifactId>assistant-agent</artifactId>
    <version>0.2.4</version>
</parent>

<!-- 修改后 -->
<parent>
    <groupId>com.sftech.agent.assistant</groupId>
    <artifactId>assistant-agent</artifactId>
    <version>0.2.4-SNAPSHOT</version>
</parent>
```

**特殊处理**:
- `assistant-agent-autoconfigure/pom.xml` 删除了独立的 `distributionManagement` 配置，继承父 POM 配置

---

### 3. Maven settings.xml 配置

**文件路径**: `~/.m2/settings.xml`

新增服务器认证配置:
```xml
<servers>
  <!-- 顺丰仓库认证配置 -->
  <server>
    <id>sf-snapshot</id>
    <username>deployment</username>
    <password>deployment123</password>
  </server>
  <server>
    <id>sf-release</id>
    <username>deployment</username>
    <password>deployment123</password>
  </server>
</servers>
```

---

## 二、发布结果

执行命令:
```bash
export JAVA_HOME=$(/usr/libexec/java_home -v 17)
mvn clean deploy -DskipTests
```

**发布成功模块**:

| 模块名 | ArtifactId | 状态 |
|--------|------------|------|
| 父项目 | `assistant-agent` | ✅ SUCCESS |
| 公共模块 | `assistant-agent-common` | ✅ SUCCESS |
| 核心模块 | `assistant-agent-core` | ✅ SUCCESS |
| 评估模块 | `assistant-agent-evaluation` | ✅ SUCCESS |
| 提示词构建器 | `assistant-agent-prompt-builder` | ✅ SUCCESS |
| 扩展模块 | `assistant-agent-extensions` | ✅ SUCCESS |
| 自动配置 | `assistant-agent-autoconfigure` | ✅ SUCCESS |
| 启动模块 | `assistant-agent-start` | ✅ SUCCESS |

**发布仓库**: http://repo.sftcwl.com/repository/sf-snapshot

---

## 三、使用方式

### 1. 作为 BOM 导入版本管理

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.sftech.agent.assistant</groupId>
            <artifactId>assistant-agent</artifactId>
            <version>0.2.4-SNAPSHOT</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### 2. 引用具体功能模块

```xml
<dependencies>
    <dependency>
        <groupId>com.sftech.agent.assistant</groupId>
        <artifactId>assistant-agent-core</artifactId>
    </dependency>
</dependencies>
```

### 3. 引用完整功能（推荐）

```xml
<dependency>
    <groupId>com.sftech.agent.assistant</groupId>
    <artifactId>assistant-agent-start</artifactId>
    <version>0.2.4-SNAPSHOT</version>
</dependency>
```

---

## 四、注意事项

1. **版本号**: 当前使用 `0.2.4-SNAPSHOT`，后续发布 Release 版本需要:
   - 修改 `pom.xml` 中的 version 为 Release 版本号（如 `0.2.4`）
   - 在父 POM 中添加 `repository` 配置指向 `sf-release`

2. **仓库认证**: 确保 `~/.m2/settings.xml` 中配置了正确的认证信息

3. **Java 版本**: 项目需要 Java 17，发布时请确保使用正确的 Java 版本:
   ```bash
   export JAVA_HOME=$(/usr/libexec/java_home -v 17)
   ```

4. **网络访问**: 发布需要能够访问 `http://repo.sftcwl.com`

---

## 五、相关文件清单

| 文件路径 | 变更类型 |
|----------|----------|
| `/Users/01428674/Ai/sds-spring-ai-project/AssistantAgent/pom.xml` | 修改 |
| `/Users/01428674/Ai/sds-spring-ai-project/AssistantAgent/assistant-agent-common/pom.xml` | 修改 |
| `/Users/01428674/Ai/sds-spring-ai-project/AssistantAgent/assistant-agent-core/pom.xml` | 修改 |
| `/Users/01428674/Ai/sds-spring-ai-project/AssistantAgent/assistant-agent-extensions/pom.xml` | 修改 |
| `/Users/01428674/Ai/sds-spring-ai-project/AssistantAgent/assistant-agent-prompt-builder/pom.xml` | 修改 |
| `/Users/01428674/Ai/sds-spring-ai-project/AssistantAgent/assistant-agent-evaluation/pom.xml` | 修改 |
| `/Users/01428674/Ai/sds-spring-ai-project/AssistantAgent/assistant-agent-autoconfigure/pom.xml` | 修改 |
| `/Users/01428674/Ai/sds-spring-ai-project/AssistantAgent/assistant-agent-start/pom.xml` | 修改 |
| `~/.m2/settings.xml` | 修改 |
