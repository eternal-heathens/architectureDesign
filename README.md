# Architecture Design

产品方案输出、技术架构输出与详细设计文档输出三类框架汇总。

## 目录

- `index.html`：总入口页
- `docs/trigger-phrases.md`：触发词和输入模板速查
- `docs/multi-role-framework.md`：三类多角色框架汇总
- `frameworks/01-product-plan/`：CEO / 产品总监 / 技术总监产品方案输出框架
- `frameworks/02-technical-architecture/`：技术总监 / 架构师 / 技术骨干技术架构输出框架
- `frameworks/03-detailed-design/`：两名资深开发 / 一名测试详细设计文档输出框架

## 统一结构

每个框架目录统一包含：

```text
index.html       静态入口页
framework.md     框架正文
templates/       输出模板，可选
assets/          页面资源，可选
```

## 触发词

```text
产品方案输出：
架构评审：
详细设计文档输出：
```

## 角色失败重试

多角色方案统一采用简单重试策略：单个角色超时或调用失败时默认重试 1 次，重试间隔 3 秒；重试仍失败时保留已完成阶段产物，并记录失败角色和失败原因。

## 使用方式

直接打开 `index.html` 查看静态页面，或阅读 `frameworks/` 和 `docs/` 下的 Markdown 文档。
