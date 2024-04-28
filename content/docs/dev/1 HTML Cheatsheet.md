---
title: "HTML Cheatsheet"
weight: 1
bookToc: true
---

### 结构 Structure

```html
<!DOCTYPE html>
<html>
<head>
    <title>Page Title</title>
</head>
<body>
    <h1>This is a Heading</h1>
    <p>This is a paragraph.</p>
</body>
</html>
```

- `<html>`: HTML页面的根节点
- `<head>`: HTML页面的meta信息，如`<title>`，样式表等
- `<body>`: HTML页面的可视内容
- `<title>`: 显示在浏览器标签页上的标题

### 文本标题 Headings

`<h1>`-`<h6>`

### 文本格式 Text Formatting

- `<p>`: Paragraph，段落
- `<br>`: Line break，换行
- `<strong>`: Strong emphasis (bold)，加粗
- `<em>`: Emphasis (italic)，斜体
- `<u>`: Underlined text，下划线

### 链接和图像 Links and Images

- `<a href="url">`: Anchor Hyperlink，超链接
- `<img src="image_path" alt="alt text">`: 图片

### 列表 Lists

- `<ul>`: Unordered list，无序列表
- `<ol>`: Ordered list，有序列表
- `<li>`: List item，列表项

### 表格 Tables

```html
<table>
    <tr>
        <th>Header 1</th>
        <th>Header 2</th>
    </tr>
    <tr>
        <td>Data 1</td>
        <td>Data 2</td>
    </tr>
</table>
```

- `<table>`: Defines a table，表格
- `<tr>`: Table row，表行
- `<th>`: Table header，表头
- `<td>`: Table cell，表项

### 表单 Forms

```html
<form action="submit_to_url">
    <label for="id1">Name:</label>
    <input type="text" id="id1" name="name">
    <input type="submit" value="Submit">
</form>
```

- `<form>`: Defines a form，表单
- `<label>`: Label for an input element，标签
- `<input>`: Input field，输入框
- `<textarea>`: Multiline text input，文本框
- `<button>`: Clickable button，按钮

### 语义标签 Semantic HTML5

这些标签提供了更明确的代码结构，并帮助搜索引擎和辅助技术更好地理解网页内容的结构和意义

- `<header>`: Introductory content or navigational links.
- `<footer>`: Footer for a section or page.
- `<section>`: Section in a document.
- `<article>`: Independent, self-contained content.
