---
type: concept
topic: 构建发布与运维排查
aliases: [Linux到Docker的交付链]
status: reference
created: "2026-10-20"
updated: "2026-10-20"
tags: [技术储备, Linux, Git, Maven, Jenkins, Docker]
related: "[[part78-technical-skills/chapter02-java/section03-jmm-jvm-and-diagnostics]]"
---

# 从源码到服务：构建、发布与排查

> 学习材料，练习未执行；资料核验日期：2026-10-20。命令与插件选项需按目标Linux、JDK、Maven、Jenkins和Docker版本确认。

## 核心解释

交付链应能回答“线上运行的内容来自哪个提交、用什么依赖构建、经过哪些验证、如何回退”。Git记录源码快照，Maven定义构建依赖与阶段，Jenkins编排执行，Docker封装运行环境；Linux负责实际进程、文件、网络与资源调度。工具名称本身不能证明交付可靠。

Git的工作区、暂存区和提交是三个不同状态：`git diff`检查未暂存差异，`git diff --cached`检查已暂存差异，`git status`还会暴露未跟踪文件。提交前需确认变更范围，不能用重置、清理或覆盖别人的工作来制造“干净工作区”。[Git官方教材](https://git-scm.com/book/en/v2/Git-Basics-Recording-Changes-to-the-Repository)

Maven的`validate → compile → test → package → verify → install → deploy`是生命周期中的阶段；执行后一个阶段通常包含之前阶段。`package`生成制品，`install`放本地仓库，`deploy`发布到远程制品仓库，不是把应用部署到生产机器。插件goal绑定阶段决定实际做什么；仅运行单元测试不等于集成验证。[Maven生命周期](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html)

Jenkins Pipeline把检出、构建、检查、归档及部署等步骤写进可评审的流水线。学习时应关注阶段输入输出、凭据范围、失败即停、制品版本及环境差异；“脚本执行成功”只表示该步骤退出码符合预期，不自动证明服务可用。[Jenkins Pipeline](https://www.jenkins.io/doc/book/pipeline/)

Docker镜像由构建步骤产生，容器是镜像的运行实例。多阶段构建可分离编译环境与运行环境，`.dockerignore`可减少构建上下文，基础镜像标签可变，应记录镜像digest和构建来源。镜像封装不能抹平CPU架构、宿主内核、外部服务和配置差异。[Docker构建实践](https://docs.docker.com/build/building/best-practices/)

## 场景：一个最小交付闭环

```text
固定提交与构建工具 → 解析并记录依赖 → 编译/测试/契约检查
→ 构建一次并标识制品 → 测试环境验证同一制品
→ 受控发布 → 就绪/业务探针与指标观察 → 保留回退路径
```

教学设计中，流水线记录提交SHA、JDK和插件版本、测试报告、镜像digest及配置版本。同一制品跨环境推广，环境配置独立管理。数据库结构升级还应检验前后版本兼容；回退旧镜像不一定能回退数据变更。

## Linux排查：沿请求路径收集证据

1. **界定故障**：何时开始、哪些接口/实例、错误率与p95是否同时变化，最近有无发布或配置切换。优先记录时间、时区和traceId。
2. **进程与资源**：用`ps`、`top`、`free`、`df`以及对应容器指标检查CPU、内存、磁盘和进程存活；注意容器cgroup限制与宿主机剩余内存不是同一口径。
3. **网络与依赖**：用`ss`看监听与连接，核查DNS、TLS、路由、连接池、下游延迟；应用500与网关502/504需要分层解释。
4. **应用证据**：按服务管理方式查看日志（如`journalctl`或容器日志），再看线程栈、GC、慢SQL和池排队。`/proc/<pid>/`可提供进程状态、映射和打开文件等信息，读取权限及字段因系统而异。[Linux /proc文档](https://docs.kernel.org/filesystems/proc.html)

排查产出应是“现象 → 证据 → 假设 → 可验证动作”，例如“CPU低而延迟高，线程栈集中等待数据库连接”比“机器慢，先重启”更能指导下一步。涉及Java运行时的细化步骤见[[part78-technical-skills/chapter02-java/section03-jmm-jvm-and-diagnostics|JVM排查]]。

## 边界与常见误区

- Git分支不能替代数据备份；提交前也要确认没有混入凭据、生成文件或无关变更。不要从日志、镜像层或流水线输出中泄露密钥。
- Maven依赖冲突和本地缓存会让“我的机器能编译”不具备可复现性；应记录有效配置和依赖树。不要默认跳过测试可以作为正式发布依据。
- 健康检查应区分进程存活与可接流量；过于激进的依赖检查可能让所有实例同时退出服务。
- Docker容器不是虚拟机；把全部数据写容器临时层会增加重建后的丢失风险。容器内存限制与JVM堆参数需共同核对。
- 重启可能暂时恢复，也会消失现场。先收集必要证据；heap dump、抓包及日志导出可能有敏感数据与磁盘成本。

## 一句话面试回答

“应围绕源码、制品、配置和运行实例建立可追溯链，流水线负责重复执行检查，线上故障则按入口、应用、依赖和资源逐层用证据定位。”

## 实操练习（未执行）

在独立练习项目画出Git提交到容器运行的流程，写一份不含生产凭据的流水线草稿。把一次依赖变更、一次测试失败、一次容器启动但未就绪分别放到正确阶段，说明哪一步应阻止发布。再用脱敏的模拟日志整理一张排查表：症状、命令/指标、可能解释、下一步证据。

验收成果：制品溯源表、流水线草稿、失败处理与回退说明、三类故障的排查记录。只有实际运行后才填写耗时、通过率与恢复效果。

**返回：** [[part78-technical-skills/chapter00-技术技能储备|技术技能储备]] | [[part00-index/part00-主页|主页]]
