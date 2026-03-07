# 工作知识地图

> 数仓开发工程师 · 工作相关的项目、技术和会议记录汇总

## 项目文档

```dataviewjs
const pages = dv.pages('"20_工作/项目文档"').sort(p => p.file.mtime, 'desc')
if (pages.length === 0) {
  dv.paragraph("*暂无笔记，新建笔记后自动出现*")
} else {
  dv.list(pages.map(p => `[[${p.file.name}]] · ${dv.date(p.file.mtime).toFormat("yyyy-MM-dd")}`))
}
```

## 技术笔记

```dataviewjs
const pages = dv.pages('"20_工作/技术笔记"').sort(p => p.file.mtime, 'desc')
if (pages.length === 0) {
  dv.paragraph("*暂无笔记，新建笔记后自动出现*")
} else {
  dv.list(pages.map(p => `[[${p.file.name}]] · ${dv.date(p.file.mtime).toFormat("yyyy-MM-dd")}`))
}
```

## 会议记录

```dataviewjs
const pages = dv.pages('"20_工作/会议记录"').sort(p => p.file.mtime, 'desc')
if (pages.length === 0) {
  dv.paragraph("*暂无笔记，新建笔记后自动出现*")
} else {
  dv.list(pages.map(p => `[[${p.file.name}]] · ${dv.date(p.file.mtime).toFormat("yyyy-MM-dd")}`))
}
```
