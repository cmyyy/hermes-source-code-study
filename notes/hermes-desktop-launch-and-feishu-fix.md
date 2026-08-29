# Desktop 启动与飞书连接修复：一个版本不匹配的教训
> [← 返回笔记地图](../README.md#笔记地图) · 故障排查与实战


> 源码位置：plugins/platforms/feishu/adapter.py → FeishuWSClient；`hermes desktop` 启动命令
> 研读版本：hermes-agent v0.19.0（commit d71033a40）

## 这是什么 / 解决什么问题

记录两件事：Hermes Desktop（Electron 桌面应用）的正确启动方式，以及启动后飞书网关连接报错的根因与修复。报错症状是 `TypeError: Client.__init__() got an unexpected keyword argument 'extra_ua_tags'`——**代码用的参数在本机安装的依赖版本里根本不存在**。这是一个典型的"代码超前于依赖版本"的排障案例。

## 为什么这么设计（启动流程与排障要点）

- **`hermes desktop` 一键构建启动**。首次运行自动安装 workspace 的 Node 依赖、用 electron-builder 打包当前 OS 的 Electron 应用（产物在 `apps/desktop/release/win-unpacked/Hermes.exe`），然后启动。常用选项分层清晰：`--source`（`electron .` 源码模式，开发调试）、`--skip-build`（跳过构建直接启动已有打包）、`--force-build`（强制完全重建）、`--build-only`（只构建不启动，`hermes update` 用）、`--cwd` / `--hermes-root`（指定初始目录和源码根）。国内网络首次构建可用 `ELECTRON_MIRROR=https://npmmirror.com/mirrors/electron/` 加速 Electron 二进制下载。

- **extra_ua_tags 的版本陷阱**。飞书 adapter 调用 `lark_oapi.ws.Client` 时传了 `extra_ua_tags=["channel"]`——这个参数用于让飞书服务器推送群 @mention 事件（Channel 协议），但该参数在 lark-oapi **1.6.8 才加入**，本机 venv 里是 1.5.3。验证方法很朴素：`inspect.signature(Client.__init__)` 看签名里有没有这个参数。

- **extra 依赖必须显式安装**。pyproject.toml 里 `feishu = ["lark-oapi==1.6.8", "qrcode==7.4.2"]`——`uv sync` 不带 `--extra feishu` 时**不会装正确版本**，这正是问题来源（venv 里残留旧解析的 1.5.3）。修复：`uv sync --extra feishu`；Windows 下文件被占用报错时用 `uv pip install "lark-oapi==1.6.8"` 直接升级。

## 关键机制/流程

排障链路：症状（TypeError: extra_ua_tags）→ 根因（代码参数 vs 依赖版本不匹配：参数 1.6.8 才引入，本机 1.5.3）→ 验证（inspect.signature 确认签名）→ 修复（按 extra 依赖锁定的版本安装）。通用教训：**飞书/矩阵等平台适配依赖必须用 `uv sync --extra <name>` 安装，报参数错误先查依赖版本与代码参数的匹配关系**。

## 相关笔记

- feishu-adapter-setup（飞书平台适配搭建）
- feishu-markdown-issue（飞书 markdown 兼容问题）
