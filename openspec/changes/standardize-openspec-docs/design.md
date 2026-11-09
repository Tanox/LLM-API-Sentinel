# 设计文档：标准化 OpenSpec 文档

> **历史归档**：本提案保留提出时的版本号、路径与命令（如 v2.5.0 / `.trae/rules/` / `/workspace/` / npm 命令），仅作历史记录，不代表 v2.10.35 的当前状态。当前事实以双语 README、`docs/` 与 `openspec/*.md` 顶部的「代码对齐要点」为准。
## 文档结构设计

### 保留的文件列表

| 文件 | 目的 |
|------|------|
| README.md | 索引文档（更新索引，中文） |
| architecture.md | 架构概览（统一语言为中文） |
| data.md | 数据模型与安全（完善内容） |
| ui.md | UI 组件库（扩展内容） |
| logic.md | 逻辑与服务（扩展内容） |
| features.md | 功能规格（重写为中文） |
| project.md | 项目规范（合并 project.md 和 project-spec.md） |

### 将删除的文件
- project-spec.md（与 project.md 合并）
- i18n.md（已无国际化框架）

### .trae/rules/ 目录整理
| 文件 | 目的 |
|------|------|
| project_rules.md | 项目规范（已有，需检查） |
| git-commit-message.md | Git 提交规范（已有，需检查） |

## 内容指南

### 文档长度
- 每个文档保持 1-2 页（约 500-1500 字

### 语言规范
1. **标题：中文
2. **代码注释：英文（代码示例中的注释可以保留英文
3. **代码：英文（代码本身保留原样

### 文档结构
```
# 标题
## 一级标题
### 二级标题
- 列表项
- 列表项
```
```
代码块
```

## 主要内容更新要点

### features.md
- 翻译为中文
- 整理版本历史为简洁的中文
- 保持功能描述更准确

### project.md
- 合并 project-spec.md 的内容
- 保留简洁概要的风格
- 整合代码规范、开发流程等内容

### .trae/rules/ 文档
- 检查 project_rules.md 的语言一致性
- 检查 git-commit-message.md 的语言一致性
- 确保与 openspec 文档风格统一

### 其他文档
- 统一语言为中文
- 补充缺失的详细信息
- 更新过时的版本信息
