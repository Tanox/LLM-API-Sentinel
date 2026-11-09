# 任务清单：标准化 OpenSpec 文档

> **历史归档**：本提案保留提出时的版本号、路径与命令（如 v2.5.0 / `.trae/rules/` / `/workspace/` / npm 命令），仅作历史记录，不代表 v2.10.35 的当前状态。当前事实以双语 README、`docs/` 与 `openspec/*.md` 顶部的「代码对齐要点」为准。
## 任务列表

### 1. 更新 features.md
- [x] 将文档完整翻译为中文
- [x] 整理版本历史，使用中文描述
- [x] 保持简洁概要风格（1-2 页）

### 2. 删除 i18n.md
- [x] 删除 i18n.md 文件
- [x] 更新 README.md 移除相关索引

### 3. 合并 project.md 和 project-spec.md
- [x] 整合两个文档的内容到 project.md
- [x] 直接删除 project-spec.md
- [x] 确保没有信息丢失
- [x] 保持简洁概要风格

### 4. 更新 architecture.md
- [x] 统一文档语言为中文
- [x] 检查并补充缺失的架构细节
- [x] 保持简洁概要风格

### 5. 更新其他 openspec 文档
- [x] 更新 data.md - 完善数据模型描述
- [x] 更新 ui.md - 补充 UI 组件细节
- [x] 更新 logic.md - 补充逻辑与服务描述

### 6. 整理 .trae/rules/ 目录
- [x] 检查 project_rules.md 语言一致性
- [x] 检查 git-commit-message.md 语言一致性
- [x] 确保与 openspec 文档风格统一

### 7. 更新 README.md
- [x] 更新文档索引
- [x] 确保所有链接有效
- [x] 添加 .trae/rules/ 目录说明

## 额外完成的优化

- [x] 清理冗余文件（app/locales/ 目录、out/ 目录）
- [x] 统一 constants 定义（删除重复的 constants.ts）
- [x] 更新项目名称（从 ai-studio-applet 改为 llm-api-sentinel）
- [x] 统一版本号（所有文件统一为 v2.5.0）
- [x] 更新 .gitignore 添加 out/ 目录

## 验收标准

- [x] 所有 openspec 文档使用中文
- [x] 所有 .trae/rules/ 文档语言一致
- [x] 文档简洁，每个 1-2 页
- [x] 无重复内容
- [x] 信息准确反映当前代码状态
- [x] 无过时信息
- [x] i18n.md 和 project-spec.md 已删除
- [x] 文档格式统一
