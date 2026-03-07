# 学习知识地图

> 技术成长 · 读书积累 · 课程学习

## 技术笔记

```dataviewjs
const pages = dv.pages('"30_学习/技术"').sort(p => p.file.mtime, 'desc')
if (pages.length === 0) {
  dv.paragraph("*暂无笔记，新建笔记后自动出现*")
} else {
  dv.list(pages.map(p => `[[${p.file.name}]] · ${dv.date(p.file.mtime).toFormat("yyyy-MM-dd")}`))
}
```

## 读书笔记

```dataviewjs
const pages = dv.pages('"30_学习/读书笔记"').sort(p => p.file.mtime, 'desc')
if (pages.length === 0) {
  dv.paragraph("*暂无笔记，新建笔记后自动出现*")
} else {
  dv.list(pages.map(p => `[[${p.file.name}]] · ${dv.date(p.file.mtime).toFormat("yyyy-MM-dd")}`))
}
```

## 课程笔记

```dataviewjs
const pages = dv.pages('"30_学习/课程笔记"').sort(p => p.file.mtime, 'desc')
if (pages.length === 0) {
  dv.paragraph("*暂无笔记，新建笔记后自动出现*")
} else {
  dv.list(pages.map(p => `[[${p.file.name}]] · ${dv.date(p.file.mtime).toFormat("yyyy-MM-dd")}`))
}
```
