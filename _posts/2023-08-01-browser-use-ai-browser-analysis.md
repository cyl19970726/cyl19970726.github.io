---
layout: post
title: "深度解析：融资1700万美元的AI浏览器Browser-Use是如何实现的"
date: 2023-08-01 09:00:00 +0800
categories: ai
tags: ai browser automation web3 agent
---

# 深度解析：融资1700万美元的AI浏览器Browser-Use是如何实现的

AI控制浏览器正成为一个热门的研究和投资领域。最近，Browser-Use以1700万美元的融资额引起了业内广泛关注。在这篇文章中，我将深入解析这款AI浏览器是如何实现的，以及其背后的核心技术原理。

## 1. AI控制浏览器的应用场景

浏览器作为人类参与互联网的重要基础设施，不仅是我们获取信息的渠道，也已深度融入我们的日常生活。可以说，人类的互联网生命很大程度上依赖于浏览器。因此，一旦LLM能够完全控制浏览器，它就可以执行几乎所有互联网上的任务。

例如一个机票预订场景：
> 用户：这是我的邮箱账号 example@gmail.com, 密码：xxxx；请帮我登陆携程网站购买一张今天北京飞往上海的机票，最便宜的，同时我还没有注册携程账号，你需要帮我用我的邮箱在携程上注册账号。

AI Agent可以通过以下步骤完成任务：
1. 分析用户意图
2. 在携程上注册账号，利用Browser Control能力收集和执行注册动作
3. 打开谷歌邮箱获取验证码
4. 返回携程完成注册
5. 搜索并购买最便宜的机票
6. 调用支付系统完成支付
7. 完成整个流程并提供确认信息

具体使用场景可参考[OpenAI的Browser-Use示例](https://openai.com/index/computer-using-agent/)或[browser-use仓库](https://github.com/browser-use/browser-use?tab=readme-ov-file#demos)中的演示。

## 2. AI控制浏览器的不同技术路线

AI Agent控制网页可分为三个基本步骤：
1. 理解用户指令（通过大模型实现）
2. 解析网页
3. 在网页上执行动作

目前有两条主流技术路线：

### 截图路线
以OpenAI-Operator为代表，通过网页截图让Agent理解页面内容，然后直接控制鼠标和键盘模拟用户操作。

![截图路线原理](/assets/images/webpage-control-screenshot.png)

### DOM路线
Browser-Use采用的路线，通过解析HTML页面的DOM树结构，标识可交互节点，然后将"任务+DOM Tree+可交互节点列表"提交给LLM，让LLM选择要交互的节点，最后通过Playwright执行交互动作。

![DOM路线原理](/assets/images/webpage-control-dom.png)

## 3. Browser-Use的解决方案

Browser-Use采用DOM解析路线，其实现方案如下：

### 3.1 解析网页

Browser-Use在浏览器端注入`buildDomTree.js`代码，该代码会：
- 从`<body>`开始递归遍历整个DOM树
- 对每个节点收集关键信息：
  - tagName、attributes、xpath、children
  - 可见性（isVisible）
  - 是否可交互（isInteractive）
  - 是否在viewport内（isInViewport）
  - 是否顶层元素（isTopElement）
  - 为可交互元素分配highlightIndex

这些属性的设计目的是让AI操作浏览器时更像人类：
- isVisible：过滤不可见元素，防止误操作
- isInteractive：只对可交互元素操作
- isInViewport：优先操作视口内元素，模拟真实用户行为
- isTopElement：避免点击被遮挡的元素

### 3.2 操作网页

Browser-Use实现了丰富的操作动作，包括：

- 基础导航：go_to_url、go_back、search_google
- 基础交互：click_element_by_index、input_text、scroll_down/up
- 表单操作：get_dropdown_options、select_dropdown_option
- 多标签管理：switch_tab、open_tab、close_tab
- 特殊功能：extract_content、save_pdf
- 高级操作：drag_drop、scroll_to_text、send_keys等
- Google Sheets专用操作：get_sheet_contents等

其中最核心的两个操作是`click_element_by_index`和`input_text`，它们通过selector_map（指定索引到DOM节点的映射）来定位和操作网页元素。

### 3.3 联系解析与操作

Browser-Use建立了一个完整的链路：

1. DOM解析 → DOM Tree + selector_map
2. LLM选择操作元素的索引index
3. Action函数通过索引从selector_map获取完整DOM节点
4. 借助DOM节点的xpath等属性执行实际浏览器操作
5. 返回操作结果，包括周围上下文文本

这种设计让LLM能通过简单的索引号精准操作网页元素，同时保留复杂交互所需的全部上下文信息。

### 3.4 专用Action

Browser-Use实现了"不同页面可以动态注册不同Action"的机制：

- Action注册是静态的，但Action可用性是动态的
- 每次step前，Agent都会根据当前页面动态生成只包含"可用Action"的ActionModel
- LLM只能选择这些Action，保证安全性和泛化能力
- 可以通过page_filter、domains等参数实现"只在特定页面暴露Action"

## 4. Browser-Use完整流程

完整的Browser-Use工作流程如下：

1. 注入BuildDomTree.js代码到当前页面
2. 提取DOM Tree和可交互节点列表
3. LLM理解用户指令
4. 将网页截图、DOM Tree和可交互节点列表提交给LLM
5. LLM选择合适的Action及参数
6. Playwright控制浏览器执行操作，生成新的页面状态
7. 循环直到任务完成

## 结语

Browser-Use通过精心设计的DOM解析和操作机制，使AI能够像人类一样交互式地使用浏览器。与截图路线相比，DOM路线提供了更结构化的网页理解能力和更精准的交互控制。

随着这类技术的不断发展，我们可以期待AI助手在未来能够帮助我们完成更多复杂的网络任务，从而节省时间并提高效率。 