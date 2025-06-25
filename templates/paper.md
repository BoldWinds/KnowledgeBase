---
DOI: <% await tp.system.prompt("请输入论文的 DOI 或 URL") %>
Description: <% await tp.system.prompt("一句话描述论文核心内容") %>
Publication Year: <% await tp.system.prompt("请输入发表年份", tp.date.now("YYYY")) %>
Rating: <% await tp.system.suggester(["⭐", "⭐⭐", "⭐⭐⭐", "⭐⭐⭐⭐", "⭐⭐⭐⭐⭐"], ["⭐", "⭐⭐", "⭐⭐⭐", "⭐⭐⭐⭐", "⭐⭐⭐⭐⭐"]) %>
Status: <% await tp.system.suggester(["📥 To Read (待读)", "👀 Reading (在读)", "✅ Done (读完)"], ["To Read", "Reading", "Done"]) %>
Tag: <% await tp.system.prompt("请输入核心标签", "Paper") %>
---

## 核心观点

## 研究方法

## 总结