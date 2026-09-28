# 仓库说明

深度研究报告集合。每份报告以独立目录组织, 统一放置于年份目录下 (如 `2026/`)。

## 命名规则

每份报告目录结构如下:

```txt
<年份>/<YYYY-MM-DD>-<english-slug>/
  <english-slug>.md          # 报告正文, 文件名与目录 slug 一致
  <资源目录>/                 # 图片、图表等附件, 目录名不限, 见下
    <图片文件>
```

### 报告目录

- 目录名格式: `YYYY-MM-DD-<english-slug>`
- `YYYY-MM-DD` 为报告成稿(发布)日期
- `english-slug` 为小写英文、单词间用短横线 `-` 连接的简短描述
- 示例: `2026-09-28-postgres-major-version-adoption`

### 正文文件

- 文件名为 `<english-slug>.md`, 与所在目录的 slug 一致
- 示例: 目录 `2026/2026-09-28-postgres-major-version-adoption/` 下为 `postgres-major-version-adoption.md`

### 图片与附件

- 报告引用的图片、图表等文件放在报告目录下, 正文以相对路径引用
- 目录名不限, 常见有 `assets/`、`images/`、`figs/`、`charts/` 等, 可自由选择或新增
- 文件类型不限 (如 `png`、`svg` 等)
- 文件名建议使用小写英文 (短横线或下划线分隔)

## 提交规范

- 新增报告使用 `docs: add <YYYY-MM-DD>-<english-slug>`
- 提交前遵循 `git-commit` 技能流程
