# 源码与安装包安全审查

审查日期：2026-11-17。

## 结论与范围

在本次审查的源码和官方 v1.0.0 安装包中，**未发现恶意代码、手机端主动联网、下载执行远程代码、收集或上传手机个人数据的实现**。

审查基线：

- 上游仓库：[Systina12/HyperOS-Assistant-Switcher](https://github.com/Systina12/HyperOS-Assistant-Switcher)。
- main 提交：[`2bc4b9e9818c797e30204ea75e4c6483e9058297`](https://github.com/Systina12/HyperOS-Assistant-Switcher/tree/2bc4b9e9818c797e30204ea75e4c6483e9058297)。
- v1.0.0 标签对应提交：`4e024793f60528e9664f3e909b8c8585e2247a4b`。
- 官方发布文件：`HyperOS-Assistant-Switcher-v1.0.0.zip`，23,306 字节。

逐文件检查了 main 的全部 18 个跟踪文件，包括 7 个 Shell 脚本、模块元数据、构建脚本、测试、GitHub Actions 和文档；另核对了发布 ZIP 的全部 13 个条目。没有把关键词搜索结果作为唯一判断依据。本报告新增时没有修改模块运行代码。

结论针对上述固定版本，不覆盖将来的更新、所有历史提交、第三方重新打包文件、Magisk / SukiSU 自身、GitHub Actions 依赖或被选中的助理应用。

## 手机端实际行为

| 检查项 | 结果 |
| --- | --- |
| 系统设置写入 | 仅 `system[user 0].long_press_power_key` 和 `global.power_button_long_press` 两项 |
| 系统默认助理 | 不写入助理角色，不指定助理包名，依赖用户已经选择的默认助理 |
| 系统信息读取 | 读取开机完成状态、MIUI / HyperOS 标记，以及检查当前执行 UID |
| 联网与上传 | 未发现网络客户端、Socket、远程服务器地址、上传接口或遥测逻辑 |
| 下载与动态执行 | 未发现下载脚本后执行、`eval`、编码载荷解码执行或远程更新配置 |
| 个人数据 | 未发现读取联系人、短信、照片、剪贴板、账号数据库或设备标识的逻辑 |
| 本地状态 | 在 `/data/adb/hyperos_assistant_switcher` 保存模式、守护配置、两项原始值及诊断日志 |
| 临时运行状态 | 在 `/dev/.hyperos_assistant_switcher` 保存操作锁和后台任务锁 |
| 二进制与系统修改 | 没有附带 APK / 原生二进制，没有替换框架、挂载系统文件或关闭 SELinux 的代码 |

具体实现可见基线版本的 [`module/common.sh`](https://github.com/Systina12/HyperOS-Assistant-Switcher/blob/2bc4b9e9818c797e30204ea75e4c6483e9058297/module/common.sh)。状态文件按纯数据读取，模式和守护参数有校验，没有将状态文件作为 Shell 代码加载。脚本会加载同一模块内的 `common.sh`，这是本地公共实现。

当目标设置返回 `null` 时，脚本会临时读取对应设置表，以区分“键不存在”和字符串 `null`。设置表保存在当前进程内存中，没有发现将整张表写入日志或上传的逻辑。本地日志记录操作结果和错误信息，必要时可能包含目标设置值或命令错误输出。

`launch_google_search` 是这里使用的系统入口设置值；该字符串本身不是网络请求，也没有调用 Google 网络接口的代码。助理被唤起后是否联网，取决于助理应用自身，超出本次模块审查范围。

## 后台任务与卸载恢复

默认在开机完成后约 0、10、30、60、120 秒检查设置，随后结束。用户显式开启 `guard` 后，才按配置间隔继续检查。没有发现隐蔽的额外后台任务。

卸载时如果系统设置服务尚未就绪，模块会复制自己的恢复脚本，并建立 `/data/adb/service.d/hyperos_assistant_switcher_restore.sh`，用于模块目录删除后恢复原始设置。恢复成功后删除此入口和恢复状态；恢复失败时保留，等待下次启动重试。这个入口属于卸载恢复逻辑，没有网络或数据上传代码；诊断日志会保留。

目录权限设为 `0700`，状态文件默认权限为 `0600`。删除操作针对模块自己的具体文件，没有发现递归删除用户数据的代码。测试用的 `HAS_STATE_DIR`、`HAS_RUN_DIR`、`HAS_RESTORE_HOOK` 可以重定向这些路径，正常安装使用上述默认位置。

## 仓库中的联网与上传

[`.github/workflows/ci.yml`](https://github.com/Systina12/HyperOS-Assistant-Switcher/blob/2bc4b9e9818c797e30204ea75e4c6483e9058297/.github/workflows/ci.yml) 在 GitHub 的构建机器上有正常网络活动：

- 使用官方 `actions/checkout` 拉取仓库、`actions/setup-python` 准备 Python。
- 使用 `apt-get` 安装 BusyBox、mksh、ShellCheck 等测试工具。
- 使用官方 `actions/upload-artifact` 上传 `dist/*` 构建产物到 GitHub Actions。

因此，“整个仓库完全没有联网或上传”并不准确；**联网和上传出现在 GitHub CI，手机安装的模块脚本没有这些实现**。CI 配置的仓库权限为 `contents: read`，没有发现自定义外传端点或上传手机日志的步骤。

Actions 使用 `@v7` 可变标签，外部依赖没有固定到完整提交 SHA。本次检查了仓库内工作流配置，没有审计这些外部 Action 的全部实现。构建脚本 `tools/build.py` 仅使用 Python 标准库在本地打包，没有下载依赖或执行网络请求。

## 发布包与源码一致性

官方 ZIP 的 SHA-256：

```text
b4fc0c6a0da7d93f29b325af5589a2756e3b34f81a020f9b42a1691af0e249c0
```

此值同时与官方 `.sha256` 文件及 GitHub 发布资产的摘要一致。

ZIP 条目名称唯一，没有绝对路径、`..` 路径穿越、符号链接或额外隐藏载荷。逐条比较后，13 个条目的内容均与 v1.0.0 标签源码完全相同。使用该标签源码重新构建的 ZIP 与官方 ZIP **逐字节相同**，SHA-256 也完全一致。

main 与 v1.0.0 标签的差异仅在 README；所有运行脚本、模块元数据、构建脚本、测试与工作流一致。因此直接从 main 构建时，文档差异会使整个 ZIP 的校验值不同，但不存在运行代码差异。

## 验证记录与限制

- 全部模块 Shell 脚本通过 `dash -n` 和 Android `/system/bin/sh -n` 语法检查。
- `shellcheck -x --shell=sh module/*.sh` 通过。
- 使用 dash 运行现有 38 项测试：全部通过。
- 使用 Android `/system/bin/sh` 运行同样 38 项测试：全部通过。
- 构建测试验证了 ZIP 布局、权限和重复构建一致性；另独立完成了官方发布包的逐文件比对和逐字节重建。

测试将系统设置、UID 检查及运行路径替换为临时目录中的模拟实现，没有安装模块或使用 root 修改真实手机设置。这次没有做安装后的网络抓包，也没有验证本机电源键是否确实唤起 Muse；静态审查和模拟测试不能保证所有 ROM 的兼容性，也不能构成对任何版本的绝对安全保证。
