# 学习知识地图

> 技术成长 · 读书积累 · 课程学习 · 行业实践

## 技术笔记

```dataviewjs
const pages = dv.pages('"30_学习/技术"')
  .filter(p => !p.file.folder.includes("行业分享") && !p.file.folder.includes("Flink") && !p.file.folder.includes("数据治理") && !p.file.folder.includes("Spark") && !p.file.folder.includes("OpenClaw"))
  .sort(p => p.file.mtime, 'desc')
if (pages.length === 0) {
  dv.paragraph("*暂无笔记，新建笔记后自动出现*")
} else {
  dv.list(pages.map(p => `[[${p.file.name}]] · ${dv.date(p.file.mtime).toFormat("yyyy-MM-dd")}`))
}
```

## Flink

```dataviewjs
const pages = dv.pages('"30_学习/技术/Flink"').sort(p => p.file.mtime, 'desc')
if (pages.length === 0) {
  dv.paragraph("*暂无笔记，新建笔记后自动出现*")
} else {
  dv.list(pages.map(p => `[[${p.file.name}]] · ${dv.date(p.file.mtime).toFormat("yyyy-MM-dd")}`))
}
```

## 数据治理

```dataviewjs
const pages = dv.pages('"30_学习/技术/数据治理"').sort(p => p.file.mtime, 'desc')
if (pages.length === 0) {
  dv.paragraph("*暂无笔记，新建笔记后自动出现*")
} else {
  dv.list(pages.map(p => `[[${p.file.name}]] · ${dv.date(p.file.mtime).toFormat("yyyy-MM-dd")}`))
}
```

## Spark

```dataviewjs
const pages = dv.pages('"30_学习/技术/Spark"').sort(p => p.file.mtime, 'desc')
if (pages.length === 0) {
  dv.paragraph("*暂无笔记，新建笔记后自动出现*")
} else {
  dv.list(pages.map(p => `[[${p.file.name}]] · ${dv.date(p.file.mtime).toFormat("yyyy-MM-dd")}`))
}
```

## 行业分享

> 来自各业务线的实时数仓、Flink、数仓建模等实践分享文章

```dataviewjs
const pages = dv.pages('"30_学习/技术/行业分享"').sort(p => p.file.mtime, 'desc')
if (pages.length === 0) {
  dv.paragraph("*暂无笔记，新建笔记后自动出现*")
} else {
  dv.list(pages.map(p => `[[${p.file.name}]] · ${dv.date(p.file.mtime).toFormat("yyyy-MM-dd")}`))
}
```

## OpenClaw

```dataviewjs
const pages = dv.pages('"30_学习/技术/OpenClaw"').sort(p => p.file.mtime, 'desc')
if (pages.length === 0) {
  dv.paragraph("*暂无笔记，新建笔记后自动出现*")
} else {
  dv.list(pages.map(p => `[[${p.file.name}]] · ${dv.date(p.file.mtime).toFormat("yyyy-MM-dd")}`))
}
```

## AI 专栏

```dataviewjs
const pages = dv.pages('"30_学习/AI专栏"').sort(p => p.file.mtime, 'desc')
if (pages.length === 0) {
  dv.paragraph("*暂无笔记，新建笔记后自动出现*")
} else {
  dv.list(pages.map(p => `[[${p.file.name}]] · ${dv.date(p.file.mtime).toFormat("yyyy-MM-dd")}`))
}
```

## 公众号文章

```dataviewjs
const pages = dv.pages('"30_学习/公众号"').sort(p => p.file.mtime, 'desc')
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
