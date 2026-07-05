# ai-md

用于存储 Markdown 格式的各类知识内容。

## 目录结构

```text
ai-md/
  docs/          知识内容正文
  templates/     Markdown 模板
  assets/        图片、附件等资源
  index.md       全局索引
```

## 内容分类

- `docs/ai/`：AI、模型、提示词、智能体相关内容
- `docs/dev/`：编程、工程、工具链相关内容
- `docs/productivity/`：效率工具、方法论、工作流
- `docs/notes/`：临时笔记、待整理内容

## 写作约定

- 文件名使用小写英文、数字和连字符，例如 `prompt-engineering.md`
- 每篇内容建议包含标题、日期、标签和正文
- 图片和附件放入 `assets/`，在 Markdown 中用相对路径引用
- 可从 `templates/note.md` 复制新建笔记
