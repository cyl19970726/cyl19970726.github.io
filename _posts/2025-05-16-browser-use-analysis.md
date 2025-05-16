---
layout: post
title: "融资融了1700万美元的AI浏览器 Browser-Use 是怎么实现的"
date: 2023-08-01 09:00:00 +0800
categories: ai
tags: browser ai llm automation
---

# 1. AI控制浏览器的应用场景
简单来说，目前浏览器作为人参与互联网的重要基础设施，不仅仅作为我们的信息获取渠道，同时也已经参与进我们的生活中，比如我们通过电商平台购物，社交平台交友等等，甚至可以说人类的互联网生命是活在浏览器上，因此LLM控制浏览器的意义就在于，一旦可以完全控制，它就可以做所有的事情。

比如用一个简单移动的大家爱用的购买机票的例子来说：
用户 ：这是我的邮箱账号 hhhxxx@gmail.com, 密码：hhhxxx ；请你帮我登陆携程的网站购买一张今天北京飞往上海的机票最便宜的，同时我还没有注册携程账号，你需要帮我用在携程上用我的邮箱注册账号。 

这个时候 Agent 就可以通过执行以下流程完成任务： 
1. 分析用户意图：
2. 先在携程上注册账号，那么它就会利用Browser Control 的能力收集携程网站所有能执行的动作并且找到了对应的Action，执行注册的动作
3.  继续分析下一步自己需要的操作是打开谷歌网站去拿到对应的注册用的验证码 
4.  于是打开谷歌邮箱并用对应的用户名、密码，登陆，拿到对应的验证码 
5. 分析下一步：返回携程网站购买机票 
6. 找到当天最便宜的机票，购买，同时调用google pay 进行付款 
7. 付款成功，用户在自己的邮箱里收到机票信息，以及登机牌总的来说，agent拥有了无缝接入这个世界的能力，同时agent之间的协作也会更加丝滑

具体的使用场景:
1. 可以参考openai opera里的[几个例子]([https://openai.com/index/computer-using-agent/])
2. 也可以参考 browse-use 仓库里列举的[场景](https://github.com/browser-use/browser-use?tab=readme-ov-file#demos)

# 2. AI控制浏览器的不同技术路线
首先，我们可以把Agent控制网页分为三个步骤：
1. 理解用户指令 
   1. 理解用户指令很简单可以利用大模型就实现了
2. 解析网页 
3. 在网页上执行动作

- 截图路线
现在有两个比较主流的技术路线，一种是openai-operator为首的，解析网页通过网页截图来让 Agent 理解，然后在网页上执行动作则通过 直接让Agent控制鼠标和键盘来实现，实现原理大概如下(图片来自openai operator)：
![openai-operaor](/assets/images/openai-operator.png)

- DOM 路线

还有另外一种路线，也是我们今天Browse-Use的路线，是通过把Html网页解析成树状结构体(DOM Tree)，因为本身Html网页其实都是一堆XML节点的嵌套，所以很好解析成树状结构; 
然后再标识出哪些节点是可交互节点，把这些这点的index提取出来，最终把 "Task+DOM Tree+可交互节点列表"一起提交给LLM，由LLM根据用户的Task选择对应要交互的节点，然后调用Playwright执行交互动作。
原理大概如下，后面我会详细解释这张图：
![DOM路线原理](/assets/images/Pasted%20image%2020250515161045.png)

# 3. 用DOM解析的路线实现 Agent Control Browse我们要解决什么问题

1. 怎么让解析网页和理解指令到执行动作衔接起来?
2. 我们要实现哪些动作?
	- 按照我们人使用浏览器的习惯，我们对浏览器的操作不外乎这几个动作：
	1. 点击
	2. 跳转
	3. 输入
	4. 等待
	5. 滚动
	这些动作的实现并不困难，因为类似 Playwright 之类的工作都可以轻松的实现之上的几个动作。
3. 怎么让AI的动作更像人?
	我们很容易根据自身的需求点击到正确的按钮，但是用DOM的方案就还要判断，诶我们现在拿到的这些可以交互节点中，哪些节点是我们当前窗口可以看见的节点，哪些节点是我们当前窗口看不见的节点，因此我们往往也需要给所有的节点加上一个 "可见性"的属性，从而让AI操作浏览器的时候更像人。
4. 怎么处理 Cookie ?

# 4. Browser-Use的解决方案
Browser-Use的解决方案如下：

![DOM路线原理](/assets/images/Pasted%20image%2020250515161045.png)
### 4.1 解析网页
解析网页涉及图上的 1️⃣ 和 2️⃣ :

首先Browse会在浏览器端注入 `buildDomTree.js` 里的代码

代码存在位置： https://github.com/browser-use/browser-use/blob/main/browser_use/dom/buildDomTree.js

代码注入位置： https://github.com/browser-use/browser-use/blob/main/browser_use/dom/service.py#L101

`buildDomTree.js` 里会把代码按照以下方式进行解析： 
- 遍历整个 DOM，从 `<body>` 开始递归所有节点。
- 对每个节点，收集：
  - tagName、attributes、xpath、children
  - 可见性（isVisible）
  - 是否可交互（isInteractive）
  - 是否在 viewport 内（isInViewport）
  - 是否顶层元素（isTopElement）
  - 如果是可交互元素，分配一个 `highlightIndex`（如 0, 1, 2...）
- 生成一个扁平的 `map`（id -> 节点信息），以及 `rootId`（根节点 id）。

#### Example 
假设有HTML网页如下：
```html
<!DOCTYPE html>
<html>
<head>
    <title>Simple Example Page</title>
</head>
<body>
    <div>
        <button id="btn1">Click Me</button>
        <input id="input1" type="text" placeholder="Enter text here">
        <span>Some text</span>
    </div>
</body>
</html>
```
我们的 buildDomTree.js 会通过 Playwright的evaluation函数把这js代码注入进浏览器里执行，然后我们就可以得到以下的Element Tree的Json数据：
```js
{
  rootId: "0",
  map: {
    "0": { // body
      tagName: "body",
      attributes: {},
      xpath: "/body",
      children: ["1"],
      // ... 其它属性
    },
    "1": { // div
      tagName: "div",
      attributes: {},
      xpath: "/body/div",
      children: ["2", "3", "4"],
      // ... 其它属性
    },
    "2": { // button
      tagName: "button",
      attributes: { id: "btn1" },
      xpath: "/body/div/button",
      children: [],
      isVisible: true,
      isInteractive: true,
      isTopElement: true,
      isInViewport: true,
      highlightIndex: 0, // 第一个可交互元素
      // ... 其它属性
    },
    "3": { // input
      tagName: "input",
      attributes: { id: "input1", type: "text" },
      xpath: "/body/div/input",
      children: [],
      isVisible: true,
      isInteractive: true,
      isTopElement: true,
      isInViewport: true,
      highlightIndex: 1, // 第二个可交互元素
      // ... 其它属性
    },
    "4": { // span
      tagName: "span",
      attributes: {},
      xpath: "/body/div/span",
      children: ["5"],
      isVisible: true,
      isInteractive: false,
      // ... 其它属性
    },
    "5": { // text node
      type: "TEXT_NODE",
      text: "Some text",
      isVisible: true,
    }
  }
}
```

然后接下来我们要把Json版本的Dom Tree转换成用Python版本的Map存储（本身数据没有变化，只是用了python的map来存储数据以及提取出当前可以被交互的节点列表 selector map）

代码位置： https://github.com/browser-use/browser-use/blob/main/browser_use/dom/service.py#L117

- 调用 `_construct_dom_tree(eval_page)`，把 JS 端的扁平 map 还原为 Python 的树状结构。
- 每个节点变成 `DOMElementNode` 或 `DOMTextNode`，父子关系用 `children` 和 `parent` 字段连接。
- **selector_map**：把所有有 `highlightIndex` 的节点收集起来，形成 `{highlightIndex: node}` 的字典。

但是这里面我们要注意以下几个tag的作用，这几个tag的存在可以让LLM操作网页更像人而且不会误触

| 属性           | 作用                                                                                   |
|----------------|----------------------------------------------------------------------------------------|
| isVisible      | 过滤不可见元素，防止误操作，辅助 LLM 理解页面布局                                      |
| isInteractive  | 只对可交互元素操作，生成 selector_map，提升泛化能力                                     |
| isInViewport   | 优先操作视口内元素，辅助滚动决策，模拟真实用户行为                                      |
| isTopElement   | 避免点到被遮挡元素，提升操作准确性，适配复杂页面                                        |

**这四个属性共同保证了 Browser Agent 的操作既"像人一样"，又"高效、准确、鲁棒"。**

然后接下来，我们需要把json数据继续解析成Python版本的DOM Tree，代码里的名字叫Element Tree;以及对应的可交互节点列表，代码里的名字叫(selector map)
#### b) selector_map
```python
selector_map = {
    0: <DOMElementNode for button id="btn1">,
    1: <DOMElementNode for input id="input1">
}
```
**这里的index代表selector_map的key, xpath 也可以在上面生成的element_tree得到对应的值。**

到这里我们，我们就将网页解析DOM Tree + 可交互列表了，也就完成了分析网页这一步； 

### 4.2操作网页
操作网页我们来看看， Browser-Use实现了哪些Action:

可以在`browser_use/controller/service.py` 里找到所有Action

| Action 名字              | 描述                          | 参数（主要）                             | 必要性/使用场景               | 与 DOM 的关系                        |
| ------------------------ | --------------------------- | --------------------------------------- | -------------------------- | ------------------------------------ |
| search_google            | 在当前标签页用 Google 搜索           | query: str                              | 信息检索、自动化搜索、导航到新内容          | 仅与页面跳转相关，非直接 DOM 操作                  |
| go_to_url                | 跳转到指定 URL                   | url: str                                | 任意网页导航、流程跳转                | 仅与页面跳转相关，非直接 DOM 操作                  |
| go_back                  | 浏览器后退                       | 无                                       | 多步流程回退、历史导航                | 仅与页面跳转相关，非直接 DOM 操作                  |
| wait                     | 等待指定秒数                      | seconds: int = 3                        | 等待页面加载、异步内容渲染、节奏控制         | 无直接关系                                |
| click_element_by_index   | 点击指定 index 的可交互元素           | index: int, xpath: str?                 | 按钮点击、链接跳转、菜单操作、表单提交等所有点击场景 | 依赖 selector_map 的 index 定位 DOM 元素    |
| input_text               | 在指定 index 的输入框输入文本          | index: int, text: str, xpath: str?      | 表单填写、登录、搜索、评论、输入等          | 依赖 selector_map 的 index 定位 DOM 元素    |
| save_pdf                 | 保存当前页面为 PDF                 | 无                                       | 页面归档、报告生成、内容保存             | 仅与页面内容相关，非直接 DOM 操作                  |
| switch_tab               | 切换到指定标签页                    | page_id: int                            | 多标签场景、并行任务、内容对比            | 仅与标签页管理相关，非直接 DOM 操作                 |
| open_tab                 | 新建标签页并打开指定 URL              | url: str                                | 多任务、并行浏览、内容分组              | 仅与标签页管理相关，非直接 DOM 操作                 |
| close_tab                | 关闭指定标签页                     | page_id: int                            | 资源释放、流程结束                  | 仅与标签页管理相关，非直接 DOM 操作                 |
| extract_content          | 抽取页面内容，按目标提取结构化信息           | goal: str, should_strip_link_urls: bool | 信息抽取、数据抓取、内容理解             | 读取整个 DOM 内容，结构化处理                    |
| scroll_down              | 页面下滚指定像素或一屏                 | amount: int?                            | 加载更多内容、滚动到目标、触发懒加载         | 操作 window/document，间接影响 DOM 可见性      |
| scroll_up                | 页面上滚指定像素或一屏                 | amount: int?                            | 返回顶部、查找内容、页面回溯             | 操作 window/document，间接影响 DOM 可见性      |
| send_keys                | 发送特殊按键（如 Enter、Esc、快捷键等）    | keys: str                               | 关闭弹窗、表格操作、快捷命令、特殊交互        | 可能影响 DOM 状态（如表单提交、弹窗关闭等）             |
| scroll_to_text           | 滚动到页面中包含指定文本的位置             | text: str                               | 查找内容、定位元素、辅助交互             | 通过文本查找 DOM 元素并滚动到其可见区域               |
| get_dropdown_options     | 获取下拉框所有选项                   | index: int                              | 表单分析、自动选择、内容理解             | 依赖 selector_map 的 index 定位 select 元素 |
| select_dropdown_option   | 选择下拉框指定文本的选项                | index: int, text: str                   | 表单填写、选项切换                  | 依赖 selector_map 的 index 定位 select 元素 |
| drag_drop                | 拖拽元素或坐标                     | element_source/target, 坐标等              | 拖拽排序、滑块验证、canvas绘图、UI交互    | 依赖 DOM 元素定位，操作鼠标事件                   |
| get_sheet_contents       | 获取 Google Sheets 整个表格内容     | 无                                       | 表格数据抓取、批量分析                | 通过键盘操作和剪贴板，间接与 DOM 交互                |
| select_cell_or_range     | 选中 Google Sheets 指定单元格或范围   | cell_or_range: str                      | 表格导航、数据定位                  | 通过键盘操作，间接与 DOM 交互                    |
| get_range_contents       | 获取 Google Sheets 指定单元格/范围内容 | cell_or_range: str                      | 表格数据抓取、批量分析                | 通过键盘操作和剪贴板，间接与 DOM 交互                |
| clear_selected_range     | 清空 Google Sheets 当前选中单元格    | 无                                       | 表格编辑、数据清理                  | 通过键盘操作，间接与 DOM 交互                    |
| input_selected_cell_text | 向 Google Sheets 当前单元格输入文本   | text: str                               | 表格编辑、数据录入                  | 通过键盘操作，间接与 DOM 交互                    |
| update_range_contents    | 批量更新 Google Sheets 单元格内容    | range: str, new_contents_tsv: str       | 表格批量编辑、自动化填表               | 通过键盘和剪贴板，间接与 DOM 交互                  |

**说明**

- **参数**：只列出最关键的参数，实际有些 action 还有可选参数。
- **必要性/使用场景**：说明该 action 在网页自动化中的典型用途。
- **与 DOM 的关系**：
  - "依赖 selector_map 的 index 定位 DOM 元素"表示该 action 需要结构化 DOM 信息来精确定位目标元素。
  - "仅与页面跳转/标签页管理相关"表示该 action 主要操作浏览器，不直接操作 DOM。
  - "通过键盘操作，间接与 DOM 交互"表示该 action 通过模拟用户输入影响页面内容。

#### Example: click_element_by_index 和 input_text 代码解析
这里我们以上面两个典型的Action click_element_by_index 和 input_text Action 来帮大家更好理解DOM Tree、Selector Map 和 DOM Tree之间的关系。

这两个函数是 browser-use 中最核心的交互 action，下面详细解析它们如何利用 DOM Tree 和 selector_map 与网页交互：
##### 1. click_element_by_index 详解

这个函数实现了"通过索引点击元素"的功能：

```python
@self.registry.action('Click element by index', param_model=ClickElementAction)
async def click_element_by_index(params: ClickElementAction, browser: BrowserContext):
    # ...省略部分代码...
```

###### 核心流程
1. **参数验证**：首先检查传入的 `index` 在 selector_map 中是否存在：
   ```python
   if params.index not in await browser.get_selector_map():
       raise Exception(f'Element with index {params.index} does not exist - retry or use alternative actions')
   ```

2. **获取 DOM 节点**：通过索引从 selector_map 中获取对应的 DOM 节点：
   ```python
   element_node = await browser.get_dom_element_by_index(params.index)
   ```

3. **特殊情况处理**：检查元素是否为文件上传器，如果是则不执行点击：
   ```python
   if await browser.is_file_uploader(element_node):
       # ...省略报错代码...
   ```

4. **执行点击**：调用 `_click_element_node` 执行实际的点击操作：
   ```python
   download_path = await browser._click_element_node(element_node)
   ```

5. **处理后果**：处理点击后可能的下载文件或新标签页：
   ```python
   if download_path:
       msg = f'💾  Downloaded file to {download_path}'
   else:
       msg = f'🖱️  Clicked button with index {params.index}: {element_node.get_all_text_till_next_clickable_element(max_depth=2)}'
   
   # 检测是否打开了新标签页
   if len(session.context.pages) > initial_pages:
       new_tab_msg = 'New tab opened - switching to it'
       # ...切换到新标签页...
   ```

##### 2. input_text 详解

这个函数实现了"在表单元素中输入文本"的功能：

```python
@self.registry.action('Input text into a input interactive element', param_model=InputTextAction)
async def input_text(params: InputTextAction, browser: BrowserContext, has_sensitive_data: bool = False):
    # ...省略部分代码...
```

###### 核心流程
1. **参数验证**：首先检查传入的 `index` 在 selector_map 中是否存在：
   ```python
   if params.index not in await browser.get_selector_map():
       raise Exception(f'Element index {params.index} does not exist - retry or use alternative actions')
   ```

2. **获取 DOM 节点**：通过索引从 selector_map 中获取对应的 DOM 节点：
   ```python
   element_node = await browser.get_dom_element_by_index(params.index)
   ```

3. **执行输入**：调用 `_input_text_element_node` 执行实际的文本输入：
   ```python
   await browser._input_text_element_node(element_node, params.text)
   ```

4. **日志记录**：记录操作日志，敏感数据不显示具体内容：
   ```python
   if not has_sensitive_data:
       msg = f'⌨️  Input {params.text} into index {params.index}'
   else:
       msg = f'⌨️  Input sensitive data into index {params.index}'
   ```

##### 3. 与 DOM Tree 和 selector_map 的关系

###### selector_map 作用
selector_map 是一个 `{highlight_index: DOMElementNode}` 的字典，它是浏览器交互的核心中间层：

1. **"翻译层"作用**：
   - 把 LLM 生成的简单数字索引（如 `index=2`）翻译成复杂的 DOM 节点对象
   - 使 LLM 不需要理解复杂的 CSS 选择器或 XPath 就能准确操作元素

2. **仅包含可交互元素**：
   - selector_map 只包含那些 `isInteractive=True` 的元素
   - 这大大简化了 LLM 的决策，让它专注于"有意义"的交互

###### DOM Tree 作用
DOM Tree 是完整网页的结构化表示，为交互提供了丰富上下文：

1. **元素定位**：
   - `browser._click_element_node` 和 `browser._input_text_element_node` 内部通过 DOM 节点的 xpath 属性定位元素
   - 完整的树结构让系统能处理 iframe 嵌套等复杂情况

2. **上下文信息**：
   - 元素的父子关系、文本内容、属性等都保存在 DOM Tree 中
   - 例如 `element_node.get_all_text_till_next_clickable_element(max_depth=2)` 获取点击的元素周围文本

3. **交互安全检查**：
   - 通过检查 `isVisible`、`isTopElement` 等属性确保交互的安全性
   - 如 `is_file_uploader` 检查防止意外触发文件选择对话框

##### 4. 交互链路总结

整个交互链路是：
1. DOM 解析 → DOM Tree + selector_map
2. LLM 选择操作元素的索引 index
3. Action 函数通过 `get_dom_element_by_index(index)` 从 selector_map 获取完整 DOM 节点
4. 借助 DOM 节点的 xpath 等属性，执行实际的浏览器操作
5. 返回操作结果，包括周围上下文文本

这种设计让 LLM 能通过简单的索引号精准操作网页元素，同时保留了复杂交互所需的全部上下文信息。

### 4.3 怎么让解析网页和操作网页联系起来
上面我们在4.2 click_element_by_index 和 input_tex的例子里已经提到了DOM Tree怎么和Action 联系起来的，在这里我们更加high level的介绍它：

![LLM与网页的交互方式](/assets/images/Pasted%20image%2020250515175943.png)
**LLM 之所以能在 params 里生成正确的 index（即 selector），确实是因为 prompt 里包含了 selector map 和/或结构化 DOM tree 的信息。**  
-  1. LLM 为什么能"知道"index？

- 在每一步 Agent 调用 LLM 之前，**会把当前页面的 selector map（所有可交互元素的 index、属性、文本等）和 DOM tree 的关键信息，作为 prompt 的一部分传递给 LLM**。
- 这样 LLM 就能"看到"页面上有哪些可交互元素、它们的 index、类型、文本、属性等，从而在 action params 里生成正确的 index。

-  2. 代码实现线索（@service.py）
关键点：消息管理和 prompt 构造
a) `step` 方法中
在 `step` 方法里，有这样一段：
```python
self._message_manager.add_state_message(state, self.state.last_result, step_info, self.settings.use_vision)
```

- 这里的 `state` 就是 `BrowserState`，包含了我们上面提到的 `element_tree` 和 `selector_map`以及 当前网页的screenshot。
- `add_state_message` 会把这些结构化信息**组织成 LLM prompt 的一部分**。

 b) `MessageManager` 相关
- `MessageManager` 负责把当前页面状态、历史操作、可用 action 等信息，**拼接成 LLM 的输入消息**。
- 具体会用到 `AgentMessagePrompt`、`SystemPrompt` 等 prompt 构造器，把 selector map、DOM tree、元素属性、index、文本等信息格式化为自然语言或结构化文本，供 LLM 理解。
 c) prompt 例子（伪代码）
LLM 看到的 prompt 可能类似于：
```
当前页面可交互元素列表：
0: button, text="提交", xpath="/body/div/button[1]"
1: input, placeholder="请输入用户名", xpath="/body/div/input[1]"
2: button, text="取消", xpath="/body/div/button[2]"
...

请根据用户指令选择合适的 index 进行操作。
```

### 4.4 专用Action 
另外一个需要注意的是 **browser-use 实现了"不同的 Page 可以动态注册不同的 Action"**，这部分逻辑主要在`agent/service.py` 
#### 一、核心机制总结
	1. **Action 注册是静态的**：所有 Action（如点击、输入、滚动等）都通过 Python 装饰器 `@action` 静态注册到 `Controller.registry`。
	2. **Action 的"可用性"是动态的**：每一步，Agent 会根据当前 Page 的特征（如 URL、DOM 结构、可交互元素等），**动态过滤/筛选出"当前页面可用的 Action"**，并动态生成对应的 ActionModel。
	3. **Prompt 里只暴露当前可用的 Action**：LLM 的 prompt 里只会暴露"当前页面可用的 Action"，这样 LLM 只能选择这些 Action。

#### 二、关键代码实现
1. Action 的注册与过滤
	- 所有 Action 都通过 `@action` 装饰器注册到 `Controller.registry`。
	- 每个 Action 可以带有 `page_filter`、`domains` 等参数，决定它在哪些页面可用。
	**例子：**
```python
@registry.action(
    name="search_google",
    description="在 Google 搜索当前 query",
    page_filter=lambda page: "google.com" in page.url
)
def search_google(...):
    ...
```
	- 这个 Action 只会在 URL 包含 `google.com` 的页面被视为"可用"。

2. 动态生成 ActionModel

在 `agent/service.py` 里，step()函数会调用：
```python
await self._update_action_models_for_page(active_page)
```
- 这会调用 `controller.registry.create_action_model(page=active_page)`，只把"当前页面可用的 Action"动态生成到 ActionModel 里。
3. LLM Prompt 只暴露可用 Action
- `get_prompt_description(page)` 只返回当前页面可用的 Action 描述，作为 prompt 的一部分。
- LLM 只能选择这些 Action 并生成参数。

#### 三、具体举例

1. 例子 1：在 Google 搜索页
	- 当前页面 URL: `https://www.google.com/search?q=python`
	- 可用 Action（被动态注册到 ActionModel）：
	  - `ClickElementAction`
	  - `InputTextAction`
	  - `search_google`（因为 page_filter 匹配）
2. 例子 2：在淘宝商品页
	- 当前页面 URL: `https://item.taobao.com/item.htm?id=123456`
	- 可用 Action：
	  - `ClickElementAction`
	  - `InputTextAction`
	  - `add_to_cart`（假设有 page_filter=lambda page: "taobao.com" in page.url）
3. 在任意页面
- 没有特殊 page_filter 的 Action（如"点击"、"输入"、"滚动"）**始终可用**。

####  四、代码片段解读

1. 动态生成 ActionModel 的关键代码

```python
async def _update_action_models_for_page(self, page) -> None:
    # 只把当前页面可用的 action 动态生成到 ActionModel
    self.ActionModel = self.controller.registry.create_action_model(page=page)
    self.AgentOutput = AgentOutput.type_with_custom_actions(self.ActionModel)
```

2. 动态过滤 Action 的实现（伪代码）

```python
def create_action_model(self, page=None):
    # 遍历所有注册的 action
    for action in self.actions:
        if action.page_filter is None or action.page_filter(page):
            # 只把可用的 action 加入到 ActionModel
            ...
```

#### 五、总结

- **Action 注册是静态的，Action 可用性是动态的。**
- 每次 step 前，Agent 都会根据当前 Page，动态生成只包含"可用 Action"的 ActionModel。
- LLM 只能选择这些 Action，保证了安全性和泛化能力。
- 你可以通过给 Action 注册时加 page_filter、domains 等参数，实现"只在特定页面暴露 Action"。

# 5. Browse-Use 完整流程
![DOM路线原理](/assets/images/Pasted%20image%2020250515161045.png)
1. 先通过在页面中注入 BuildDomTree.je的代码
2. 通过注入的代码提取出当前的Html页面的 Dom Tree，以及对应的可交互节点列表
3. LLM 理解用户指令
4. Html网页的截图以及Dom Tree和可交互节点列表一起提交给 LLM
	- LLM 看到的 prompt 可能类似于：
```
	当前的页面截图：
	website.png
	
	当前的DOM Tree:
	Element Tree
	
	当前页面可交互元素列表：
	0: button, text="提交", xpath="/body/div/button[1]"
	1: input, placeholder="请输入用户名", xpath="/body/div/input[1]"
	2: button, text="取消", xpath="/body/div/button[2]"
	...
	
	用户指令：
		帮我购买机票
		
	请根据用户指令、可交互元素列表、以及页面截图，选择合适的Action操作当前页面
```
5. LLM 选择合适的 Action进行执行
6. 最终由 Playwright 控制浏览器通过LLM选出的Action和对应的参数操作浏览器，并且会生成新的网页DOM Tree继续进入下一个循环直到用户的任务被完全解决