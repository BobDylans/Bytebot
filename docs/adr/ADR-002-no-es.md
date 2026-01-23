# ADR-002: 不引入 ES/OpenSearch，先用 PG FTS

## 背景
MVP 需要最少组件。

## 选项
- PG FTS
- ES/OpenSearch

## 取舍
PG FTS 简单但功能有限。

## 结论
MVP 使用 PG FTS。

## 后果
后续可引入搜索引擎。