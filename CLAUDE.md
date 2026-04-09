# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

咖啡点单微信小程序，后端基于 FastAPI，前端基于 uni-app。独立精品咖啡馆，单店模型。

**平台分离**：小程序仅服务消费者（C端），网页版服务店员（S端）和管理员（A端）。S端和A端共用网页版，登录后根据角色跳转对应页面。

## 技术栈

- **后端**: Python + FastAPI
- **C端前端**: uni-app（Vue3）→ 微信小程序
- **S/A端前端**: uni-app（Vue3）→ H5 网页版
- **部署**: Docker + Docker Compose

## 项目结构

```
project/
  ├── backend/
  │   ├── src/
  │   ├── tests/
  │   ├── pyproject.toml
  │   └── uv.lock
  ├── frontend/
  │   └── src/
  ├── docs/
  │   └── prd/           # 产品需求文档
  └── scripts/
```

