---
layout: post
title: ocw.mit.edu 漫游备忘
tags: self-learning math ocw
mathjax: true
---

这本应该是一个存在 OneNote 里的笔记，但是我写这篇文章的时候还没有大规模使用 OneNote，考虑到其他朋友有可能会文，所以就直接留在这里了。之前为了写文章凑了一组文字，现在我尽量保持简洁。

网站一共分三个，分别是 [MIT OpenCourseWare], [MIT EECS学院网站] 和 [MIT Degree Chart]。

## OCW
对于公开课网站，没什么好说的。其中的课程号也是根据学院来划分的。

其中课程编号规则如下：`DepartmenID.CourseID`
* 18.xxxx = Department of Mathematics，属于 GIR 的教学内容。<br/>18-C 是 Bachelor of Science in Mathematics with Computer Science
*  6.xxxx = Department of EECS，也是我常用的学号。

今年好像课程号升级到了4位，可以再学院网站里看到详细介绍。

* 课程上标后缀：上标¹ 表示春季授课，上标² 表示秋季授课。
* 课程号后缀 **A**：Alternative，适用于高中就上过微积分的学生，授课时间短。主要用于复习。比如18.03A
* 课程号后缀出现额外数字：比如 18.02**2** - 数字后缀一般都是更难或者有试验，物理课程类似。<br/>Additional material is included on geometry, vector fields, and linear algebra*
* 课程号带有后缀 **[J]**："Joint" 表示由多个部门联合授课的意思。一般是同一门课程，但是有多个不同的课号。


曾经有一个关注课程之间前后依赖关系的星座图，但是随着更新找不到了。他的原始出处在这里，很方便的查询课程依赖关系。推荐自学前进行查询：

https://mapping.mit.edu/curriculum-mapping


## MIT Course Catalog
本网站主要是提供的是 MIT 学生能从专业计划毕业的所有学习科目和培养计划。详细介绍了每个学校不同专业的细节。本科、硕士的培养计划、课程信息、辅修学位（Minor in Computer Science）

**GIR** - General Institute Requirement。主要包括数学、物理、化学、生物、体育。当然也包括人文艺术社科和沟通要求。[http://catalog.mit.edu/mit/undergraduate-education/general-institute-requirements/](http://catalog.mit.edu/mit/undergraduate-education/general-institute-requirements/)

**Departmental Program**：学院的课程计划，根据递进关系包括学院的公共必修课、CS专业课程、高级本科科目，和自主探索科目。[http://catalog.mit.edu/degree-charts/computer-science-engineering-course-6-3/](http://catalog.mit.edu/degree-charts/computer-science-engineering-course-6-3/)

> 早些年浏览学院网站的时候，一直不明白*电气工程和计算机科学（6-2）*，和 *计算机工程和科学（6-3）* 两个专业的区别在哪里。找了好久毫无头绪，顺便这次也在新版网站得到了答案。

## EECS 学院网站

### 学科要求

整个大学一共分为四个阶段，我们应该认为这是循序渐进的过程。

**介绍性课程*Introductory Subjects***：这些课程提供了一个机会，让学生们使用EECS领域中基础性的原则，去**识别**、**建模（*formulate*）**、**解决**实际的工程问题。
* 第一年学的主要目的是能获得基本的编程技巧、和扎实的数学基础。
* 同时让学生们能明智的发现并选择自己喜欢的基础学科。

**基础学科*Foundation Subjects***：旨在为学生们提供的接下来的十年内都很好的服务于（ *serve students well*）学生们的基础专业知识。

**前沿科目*Head Subjects***：参与 3~4 个前沿科目的学习。
* 由基础学科引出，帮助学生们在某一个领域构建更深入的认知（ *build depth*）。
* 为未来的硕士课程、雇主们感兴趣的专业领域提供一个坚实的 *Solid Credential*。

**高级本科科目*AUSes***：x2。旨在提供必要的基础知识，去发展自己感兴趣专业领域（一般都是最前沿的）。

根据不同的主修课程，有可能会要求参加一些 *Department LAB*，自主探究（*Independent Inquiry*）科目，或者在院内达到一定的知识广度。

### 术语 Abbr.

* SB Degree - Bachelor of Science (in Mathematic) 理学学士学位
* B.E, B.Eng,  B.A.I - Bachelor of Engineering 工学学士学位
* B.A.I 是拉丁名字：*Bachelor in the Art of Engineering*

在课程要求中的缩写：
* G.I.R.   - General Institute Requirements，指公共必修课。
* HASS     - Humanities, Arts, and Social Sciences，人文、艺术、社会科学。
* CI-H     - Communication-Intensive 目的在于沟通要求，可能是培养表达的课程
* REST     - Restricted Electives in Science and Technology，指的专业限选课。
* AUS      - Advanced Undergraduate Subject 高级本科科目，就是网站里列出的课程集合。
* UR / UAR - Undergraduate (Advanced) Research，目的主要是培养沟通能力。（有研讨会的课程）
* IAP      - Independent Activities Period，课程可以在 http://student.mit.edu/catalog/m6a.html 找到。
> The Independent Activities Period (IAP) takes place for four weeks in January and is an integral part of the Institute's educational program. 
>
> Students, faculty, staff, and other members of the MIT community organize about 600 noncredit activities and 100 for-credit subjects. These are publicized on the IAP website, beginning in October. 
> In addition, there are many individually arranged projects that are not officially publicized.

## MIT Unit 课时说明

MIT 使用 Units 作为课程时长的计量单位。MIT一般使用 (A-B-C)的方式来计量课程时间。

* A units = Lecture 课堂时间 
* B units = laboratory, design,  field word, 实验时间
* C units = outside preparation, 作业时间

官方给出的换算公式是： `1 Units = 14hour`， `3 Units = 1 Semester Hour`。实际上学生们会把 units 当做一个周的学习耗时。通常 MIT会将一个 12 Units 的课程的设置为 (5-0-7) or (4-0-8)，这意味一门 12 units 的课程，大概会有 4 units 的授课时间。一般来说美国一个学期有三个月左右，大概是十四个星期。 换算过来 一门 12units 的课程在一个学期内学完，好是 12 h/w。`12units * 14hour/unit = 14weeks * 12 hou/week`

* 从对应关系上，如果按照每节课 2 hours时算，每周 2 节课，三个月就能学完一门课程。
* 4小时听课时间需要8小时写作业时间，写作业是听课的1.5~2倍时长。
* 一般来说，MIT的学生每一年只会选 3~4 门课。



[MIT OpenCourseWare]: https://ocw.mit.edu
[MIT EECS学院网站]: https://www.eecs.mit.edu/
[MIT Degree Chart]: https://catalog.mit.edu/degree-charts/


