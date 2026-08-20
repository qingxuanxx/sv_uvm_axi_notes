# 《Cracking Digital VLSI Verification Interview》学习笔记

## 整理原则

本目录以原书的章节、小节和提问顺序为主线，将英文问答整理为面向数字 IC 验证初学者的中文学习笔记。

每道题尽量使用以下结构：

```text
Qn. 中文题意（必要时保留英文术语）
  → 原书答案整理：忠于原书主旨的中文转述
  → 代码/推导：保留必要过程并增加中文注释
  → 补充说明：工程实践、记忆方法或现代工具用法
  → 原书纠错：单独标记原书中的过时命令、笔误或不严谨表述
```

> 说明：本笔记不复制原书全文和原始插图。英文原题以中文题意为主，答案采用结构化转述；必要的电路图、时序图和表格将使用可独立编辑的形式重绘。

## 原书结构与笔记对应

| 原书部分 | 原书编号问题数 | 笔记 |
|---|---:|---|
| Preface / Career / Introduction / Interview Preparation / Leader Interviews | — | [00_前言与面试准备.md](00_前言与面试准备.md) |
| Chapter 1: Digital Logic Design | 50 | [01_数字逻辑设计.md](01_数字逻辑设计.md) |
| Chapter 2: Computer Architecture | 44 | [02_计算机体系结构.md](02_计算机体系结构.md) |
| Chapter 3: Programming Basics | 96 | [03_编程基础.md](03_编程基础.md) |
| Chapter 4: Hardware Description Languages | 86 | [04_硬件描述语言.md](04_硬件描述语言.md) |
| Chapter 5: Fundamentals of Verification | 18 | [05_验证基础.md](05_验证基础.md) |
| Chapter 6: Verification Methodologies | 151 | [06_验证方法学.md](06_验证方法学.md) |
| Chapter 7: Version Control Systems | 41 | [07_版本控制.md](07_版本控制.md) |
| Chapter 8: Logical Reasoning/Puzzles | 20 | [08_逻辑推理与谜题.md](08_逻辑推理与谜题.md) |
| Chapter 9: Non-Technical and Behavioral Questions | 34 | [09_非技术与行为面试.md](09_非技术与行为面试.md) |
| Closing Remarks | — | [09_非技术与行为面试.md](09_非技术与行为面试.md) |

## 标记约定

- `Q1`、`Q2` 等为原书在当章中的问题顺序。
- `Q14～Q15` 表示两道高度相关的原题在笔记中合并讲解，但不改变原题先后关系。
- `> 补充说明` 表示非原书内容。
- `> 原书纠错` 表示为了技术正确性对原文进行的修正。
- 各章末的学习重点和速查表属于笔记附加内容，不属于原书问答。
