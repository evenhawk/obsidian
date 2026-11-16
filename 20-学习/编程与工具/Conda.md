---
title: Conda
tags:
  - conda
  - python
  - 环境管理
status: maintained
updated: 2026-11-16
---

# Conda

**介绍:**

Conda是一个跨平台的环境管理器,可帮助管理Python包和Python虚拟环境.

**下载地址:**

1. GitHub: <https://github.com/conda-forge/miniforge/releases>(内网下载)
2. 各种镜像源搜索conda下载miniconda3(自己电脑下载) 公司下了会侵权,个人学习使用

**安装包:**

`Miniforge3-24.11.3-2-Windows-x86_64.zip`

**基础命令:**

| 操作 | 命令 |
| --- | --- |
| 初始化命令行 | `conda init` |
| 创建虚拟环境 | `conda create -n <name> python=<版本号>` |
| 查看当前虚拟环境列表 | `conda env list` |
| 激活虚拟环境 | `conda activate <name>` |
| 退出虚拟环境 | `conda deactivate` |
| 删除虚拟环境 | `conda env remove -n <name>` |
