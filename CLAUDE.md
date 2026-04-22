# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

这是一个基于 Docsify 的静态文档站点，记录了各 ARM SoC 平台的嵌入式开发心得（Ambarella CV72、HiSilicon hi3559、Rockchip RK3588/RK3576、Sigmastar 268g）。

## 架构

- **入口文件**: [index.html](index.html) - Docsify 配置，包含搜索、分页、LaTeX、Gitalk 评论、代码高亮等插件
- **导航配置**: [_sidebar.md](_sidebar.md) 定义侧边栏目录
- **文档内容**: [docs/](docs/) 目录下按平台分类的 Markdown 文件

## 使用方式

这是静态站点，无需构建。本地预览：

```bash
# 使用 Python 内置 HTTP 服务器
python -m http.server 3000

# 或使用 docsify CLI
docsify serve .
```

编辑 `docs/*/ch1.1.md` 中的 markdown 文件即可添加内容，侧边栏会自动链接。

## 平台文档

各平台目录包含开发笔记：

- `docs/cv72/` - 安霸 CV72 视频编码（test_encode 工具、Lua 管线配置）
- `docs/hi3559/` - 海思平台
- `docs/rk3588/` - 瑞芯微 RK3588
- `docs/rk3576/` - 瑞芯微 RK3576
- `docs/268g/` - 晨星 268g