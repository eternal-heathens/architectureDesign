# Architecture Design

技术架构方案、产品方案输出框架与多角色协作方案汇总。

## 目录

- `index.html`：总入口页
- `multi-role-output-frameworks/final-framework.md`：产品方案输出、技术架构输出、详细设计文档输出三类多角色框架汇总
- `product-to-delivery-framework/`：CEO / 产品总监 / 技术总监产品方案输出框架
- `message-broadcast-architecture/`：消息广播技术架构方案与三角色技术架构输出框架
- `detailed-design-output-framework/`：两名资深开发 / 一名测试详细设计文档输出框架

## 触发词

```text
产品方案输出：
架构评审：
详细设计文档输出：
```

## 角色失败重试

多角色方案统一采用简单重试策略：单个角色超时或调用失败时默认重试 1 次，重试间隔 3 秒；重试仍失败时保留已完成阶段产物，并记录失败角色和失败原因。

## 使用方式

直接打开 `index.html` 查看静态页面，或阅读各目录下的 Markdown 文档。
