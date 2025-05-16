---
layout: post
title: "Arc 框架解析"
date: 2024-07-18 10:05:00 +0800
categories: arc ai
tags: [agent, arc, analysis, architecture, rust, rag, pipeline, extractor]
---

## 1. 实现了 RAG 数据库

![](/assets/images/Pasted image 20250130133820.png)
### RAG 系统的完整组件

1. **EmbeddingsBuilder** - 向量生成器
```rust
// 只负责将文档转换为向量
let embeddings = EmbeddingsBuilder::new(embedding_model.clone())
    .documents(documents)?
    .build()
    .await?;
```
作用：
- 接收原始文档
- 提取文本内容
- 生成向量嵌入
- 返回文档-向量对

2. **VectorStore** - 向量存储
```rust
// 实际存储向量和文档的地方
let vector_store = InMemoryVectorStore::from_documents(embeddings);
```
作用：
- 存储文档内容
- 存储向量表示
- 提供检索接口

3. **VectorStoreIndex** - 向量索引
```rust
// 提供高效的向量检索能力
let index = vector_store.index(embedding_model);
```
作用：
- 构建向量索引
- 提供相似度搜索
- 优化检索性能

### 完整的 RAG 流程

```rust
// 1. 准备文档
let documents = vec![
    WordDefinition { /* ... */ },
    WordDefinition { /* ... */ },
];

// 2. 生成向量嵌入
let embeddings = EmbeddingsBuilder::new(embedding_model.clone())
    .documents(documents)?
    .build()
    .await?;

// 3. 创建向量存储
let vector_store = InMemoryVectorStore::from_documents(embeddings);

// 4. 创建检索索引
let index = vector_store.index(embedding_model);

// 5. 创建 RAG agent
let rag_agent = openai_client
    .agent("gpt-4")
    .preamble("System prompt...")
    .dynamic_context(1, index)  // 使用索引进行检索
    .build();
```

### 各组件的职责

1. **EmbeddingsBuilder**
```rust
pub struct EmbeddingsBuilder<M: EmbeddingModel, T: Embed> {
    model: M,
    documents: Vec<(T, Vec<String>)_,
    batch_size: Option<usize>,
    normalized: bool,
}
```
- 文档预处理
- 向量生成
- 批处理优化

2. **VectorStore**
```rust
pub trait VectorStore {
    fn add_document(&mut self, doc: Document, embedding: Embedding);
    fn get_document(&self, id: &str) -> Option<&Document>;
    // ...
}
```
- 文档存储
- 向量存储
- 基本检索

3. **VectorStoreIndex**
```rust
pub trait VectorStoreIndex: Send + Sync {
    async fn top_n<T>(&self, query: &str, n: usize) 
        -> Result<Vec<(f64, String, T)>, VectorStoreError>;
}
```
- 向量检索
- 相似度计算
- 结果排序

### 总结

完整的 RAG 数据库需要：
1. EmbeddingsBuilder - 向量生成
2. VectorStore - 存储层
3. VectorStoreIndex - 检索层
4. Agent - 应用层

这些组件共同工作才构成了完整的 RAG 系统。


## 2. 支持Static Tool 和 Dynamic Tool


在 `agent.rs` 中，Agent 结构体定义了静态和动态工具：

```rust
pub struct Agent<M: CompletionModel> {
    model: M,
    preamble: String,
    static_context: Vec<Document>,
    static_tools: Vec<String>,  // 静态工具名称列表
    temperature: Option<f64>,
    max_tokens: Option<u64>,
    additional_params: Option<serde_json::Value>,
    dynamic_context: Vec<(usize, Box<dyn VectorStoreIndexDyn>)_,
    dynamic_tools: Vec<(usize, Box<dyn VectorStoreIndexDyn>)_,  // 动态工具索引
    tools: ToolSet,  // 所有工具的集合
}
```

工具的处理主要在 completion 方法中：

```rust
impl<M: CompletionModel> Agent<M> {
    async fn completion(
        &self,
        prompt: &str,
        chat_history: Vec<Message>,
    ) -> Result<CompletionRequestBuilder<M>, PromptError> {
        let mut builder = self.model.completion_request(prompt);

        // 1. 添加系统 prompt
        builder = builder.preamble(&self.preamble);

        // 2. 添加静态工具
        // 静态工具直接从 tools 中获取已注册的工具名称
        for toolname in &self.static_tools {
            if let Some(tool) = self.tools.get(toolname) {
                builder = builder.tool(tool.definition(prompt.to_string()).await);
            }
        }

        // 3. 添加动态工具
        // 对每个动态工具索引
        for (sample_size, index) in &self.dynamic_tools {
            // 使用向量检索找到相关工具
            let similar = index.top_n_ids(prompt, *sample_size).await?;
            
            // 将找到的工具添加到 builder
            for (_, toolname) in similar {
                if let Some(tool) = self.tools.get(&toolname) {
                    builder = builder.tool(tool.definition(prompt.to_string()).await);
                }
            }
        }

        // 4. 设置其他参数
        if let Some(temperature) = self.temperature {
            builder = builder.temperature(temperature);
        }
        // ...

        Ok(builder)
    }
}
```

工具的注册是在 AgentBuilder 中完成的：

```rust
impl<M: CompletionModel> AgentBuilder<M> {
    // 添加静态工具
    pub fn tool(mut self, tool: impl Tool + 'static) -> Self {
        let toolname = tool.name();
        self.tools.add_tool(tool);
        self.static_tools.push(toolname);
        self
    }

    // 添加动态工具
    pub fn dynamic_tools(
        mut self,
        sample: usize,
        dynamic_tools: impl VectorStoreIndexDyn + 'static,
        toolset: ToolSet,
    ) -> Self {
        self.dynamic_tools.push((sample, Box::new(dynamic_tools)));
        self.tools.add_tools(toolset);
        self
    }
}
```

主要区别：

1. **静态工具**
- 通过 `tool()` 方法直接注册
- 工具名称保存在 `static_tools` 列表中
- 每次请求都会包含在 prompt 中
- 不需要向量检索

2. **动态工具**
- 通过 `dynamic_tools()` 方法注册
- 需要提供向量索引和采样数量
- 基于用户输入进行向量检索
- 只有相关的工具会被包含在 prompt 中

使用示例：

```rust
// 创建 Agent
let agent = openai_client
    .agent("gpt-4")
    .preamble("You are a helpful assistant")
    // 添加静态工具
    .tool(BasicTool::new())
    // 添加动态工具
    .dynamic_tools(2, tool_index, dynamic_toolset)
    .build();

// 使用 Agent
let response = agent.prompt("Help me calculate something").await?;
```

这种设计允许：
1. 核心工具始终可用（静态）
2. 特定工具按需发现（动态）
3. 通过向量检索实现智能工具选择
4. 优化 prompt 长度，只包含相关工具


## 3. 支持 Dynamic Context 和 Static Context


在 Rig 框架中，`static_context` 和 `dynamic_context` 的区别类似于静态工具和动态工具的概念。让我们详细分析：

### 1. 在 Agent 结构体中的定义

```rust
pub struct Agent<M: CompletionModel> {
    model: M,
    preamble: String,
    static_context: Vec<Document>,  // 静态上下文文档
    static_tools: Vec<String>,
    // ...
    dynamic_context: Vec<(usize, Box<dyn VectorStoreIndexDyn>)_,  // 动态上下文索引
    dynamic_tools: Vec<(usize, Box<dyn VectorStoreIndexDyn>)_,
    tools: ToolSet,
}
```

### 2. 在 AgentBuilder 中的添加方法

```rust
impl<M: CompletionModel> AgentBuilder<M> {
    // 添加静态上下文
    pub fn context(mut self, doc: impl Into<String>) -> Self {
        self.static_context.push(Document {
            id: format!("doc_{}", self.static_context.len()),
            text: doc.into(),
            additional_props: HashMap::new(),
        });
        self
    }

    // 添加动态上下文
    pub fn dynamic_context(
        mut self,
        sample: usize,
        dynamic_context: impl VectorStoreIndexDyn + 'static,
    ) -> Self {
        self.dynamic_context.push((sample, Box::new(dynamic_context)));
        self
    }
}
```

### 3. 主要区别

1. **静态上下文 (static_context)**
   - 始终包含在每个请求中
   - 直接作为文档存储
   - 不需要向量检索
   - 适用于基础/通用知识
   - 通过 `.context()` 方法添加

```rust
let agent = openai_client
    .agent("gpt-4")
    .context("This is a basic rule that always applies")
    .dynamic_context(3, knowledge_base_index)  // 每次检索3个最相关文档
    .build();
```

2. **动态上下文 (dynamic_context)**
   - 基于用户输入动态检索
   - 存储在向量数据库中
   - 需要向量相似度搜索
   - 适用于大规模/特定知识库
   - 通过 `.dynamic_context()` 方法添加，需要指定采样数量

```rust
let agent = openai_client
    .agent("gpt-4")
    .dynamic_context(3, knowledge_base_index)  // 每次检索3个最相关文档
    .build();
```

### 4. 在请求处理中的使用

```rust
impl<M: CompletionModel> Agent<M> {
    async fn completion(
        &self,
        prompt: &str,
        chat_history: Vec<Message>,
    ) -> Result<CompletionRequestBuilder<M>, PromptError> {
        let mut builder = self.model.completion_request(prompt);

        // 添加静态上下文 - 直接包含所有文档
        for doc in &self.static_context {
            builder = builder.context(doc);
        }

        // 添加动态上下文 - 通过向量检索找到相关文档
        for (sample_size, index) in &self.dynamic_context {
            let similar = index.top_n_ids(prompt, *sample_size).await?;
            for (_, doc_id) in similar {
                if let Some(doc) = self.get_document(&doc_id) {
                    builder = builder.context(doc);
                }
            }
        }

        // ... 其他处理
        Ok(builder)
    }
}
```

### 5. 使用场景

1. **静态上下文适用于**:
   - 系统规则和约束
   - 基本指令和说明
   - 始终需要的背景信息
   - 数量较少的固定知识

2. **动态上下文适用于**:
   - 大规模知识库
   - 特定领域的专业知识
   - 需要根据查询动态检索的信息
   - 上下文相关的参考资料

### 6. 完整使用示例

```rust
// 创建向量存储和索引
let embeddings = EmbeddingsBuilder::new(embedding_model)
    .documents(knowledge_base)?
    .build()
    .await?;

let vector_store = InMemoryVectorStore::from_documents(embeddings);
let index = vector_store.index(embedding_model);

// 创建 Agent
let agent = openai_client
    .agent("gpt-4")
    .preamble("You are a helpful assistant")
    // 添加静态上下文
    .context("Always be polite and professional")
    .context("Follow these basic guidelines...")
    // 添加动态上下文
    .dynamic_context(3, index)  // 每次检索3个最相关文档
    .build();
```

这种设计的优点：
1. 灵活性：可以同时使用固定和动态的上下文
2. 效率：只检索相关的动态上下文
3. 可扩展性：支持大规模知识库
4. 智能性：基于语义相似度的上下文检索

这种机制让 Agent 能够在保持基本行为准则的同时，根据具体查询提供相关的专业知识。

## 4. Pipeline 


Pipeline 模块的设计采用了操作符组合模式，这种架构设计主要是为了支持灵活的数据处理流程。让我详细解释这个架构：

### 1. 核心抽象

1. **基础操作 Trait (Op)**
```rust
pub trait Op: Send + Sync {
    type Input: Send + Sync;
    type Output: Send + Sync;
    
    fn call(&self, input: Self::Input) -> impl Future<Output = Self::Output> + Send;
}
```

2. **可失败操作 Trait (TryOp)**
```rust
pub trait TryOp: Send + Sync {
    type Input: Send + Sync;
    type Output: Send + Sync;
    type Error: Send + Sync;
    
    fn try_call(&self, input: Self::Input) 
        -> impl Future<Output = Result<Self::Output, Self::Error>> + Send;
}
```

### 2. 架构组件

1. **基本操作符 (op.rs)**
```rust
// 映射操作
pub struct Map<Op, F> {
    op: Op,
    f: F,
}

// 链式操作
pub struct Chain<Op1, Op2> {
    op1: Op1,
    op2: Op2,
}
```

2. **并行操作 (parallel.rs)**
```rust
pub struct Parallel<Op1, Op2> {
    op1: Op1,
    op2: Op2,
}
```

3. **条件操作 (conditional.rs)**
```rust
pub struct Conditional<T, Ops> {
    ops: Ops,
    _t: PhantomData<T>,
}
```

### 3. 设计原因

1. **组合性 (Composability)**
```rust
// 可以灵活组合不同操作
let pipeline = pipeline::new()
    .map(preprocess)           // 预处理
    .then(validate)            // 验证
    .and_then(async_process)   // 异步处理
    .map_err(handle_error);    // 错误处理
```

2. **并行处理**
```rust
let pipeline = pipeline::new()
    .chain(parallel!(
        text_process,      // 文本处理
        image_process,     // 图像处理
        metadata_process   // 元数据处理
    ));
```

3. **类型安全**
```rust
// 编译时类型检查
let pipeline: Pipeline<String, Vec<u8>> = pipeline::new()
    .map(|s: String| s.into_bytes())
    .map(|v: Vec<u8>| process_bytes(v));
```

### 4. 实际应用场景

1. **RAG 流水线**
```rust
let rag_pipeline = pipeline::new()
    // 并行执行查询和文档检索
    .chain(parallel!(
        passthrough(),                    // 保持原始查询
        lookup::<_, _, Document>(index, 5) // 检索相关文档
    ))
    // 组合查询和文档
    .map(|(query, docs)| format!("Query: {query}\nContext: {docs}"))
    // 发送到 LLM
    .prompt(llm_model);
```

2. **数据处理流水线**
```rust
let process_pipeline = pipeline::new()
    .map(parse_input)          // 解析输入
    .and_then(validate_data)   // 验证数据
    .map_ok(transform_data)    // 转换数据
    .or_else(handle_error);    // 错误处理
```

### 5. 为什么选择这种架构

1. **灵活性**
- 支持同步和异步操作
- 可以动态构建处理流程
- 易于添加新的操作类型

2. **可维护性**
```rust
// 每个操作都是独立的，易于测试和维护
#[cfg(test)]
mod tests {
    #[test]
    fn test_map_operation() {
        let op = map(|x: i32| x + 1);
        assert_eq!(op.call(1).await, 2);
    }
}
```

3. **性能考虑**
- 支持并行处理
- 最小化内存拷贝
- 异步操作优化

4. **错误处理**
```rust
// 完整的错误处理链
let pipeline = pipeline::new()
    .map(|x| Ok::<_, Error>(x + 1))
    .and_then(validate)
    .or_else(|error| async move {
        log::error!("Error: {}", e);
        Err(e)
    });
```



在 Agent 中，Pipeline 主要用于构建复杂的 RAG (Retrieval Augmented Generation) 和数据处理流程。让我通过实际场景来说明：

### 1. RAG 场景

```rust
// 基于 rag.rs 示例
let rag_pipeline = pipeline::new()
    // 1. 并行执行：保留原始查询 + 检索相关文档
    .chain(parallel!(
        passthrough(),                        // 保持原始用户查询
        agent_ops::lookup::<_, _, Document>(  // 从向量库检索文档
            index,      // 向量索引
            5          // 检索前5个最相关文档
        )
    ))
    // 2. 组合查询和上下文
    .map(|(query, docs)| {
        format!(
            "User Query: {}\n\nRelevant Context:\n{}",
            query,
            docs.iter()
                .map(|doc| doc.text.clone())
                .collect::<Vec<_>>()
                .join("\n")
        )
    })
    // 3. 发送到 LLM 生成回答
    .prompt(llm_model);
```

### 2. 多模型协作场景

```rust
let multi_model_pipeline = pipeline::new()
    // 1. 并行处理不同类型的输入
    .chain(parallel!(
        // 文本处理分支
        text_processor
            .map(process_text)
            .prompt(text_model),
        
        // 图像处理分支
        image_processor
            .map(process_image)
            .prompt(vision_model)
    ))
    // 2. 合并结果
    .map(|(text_result, image_result)| {
        format!("Text Analysis: {}\nImage Analysis: {}", 
            text_result, image_result)
    })
    // 3. 最终总结
    .prompt(summary_model);
```

### 3. 信息提取场景

```rust
// 基于 multi_extract.rs
let extraction_pipeline = pipeline::new()
    // 1. 文档预处理
    .map(preprocess_document)
    // 2. 并行提取不同类型的信息
    .chain(parallel!(
        extract::<PersonInfo>(person_extractor),
        extract::<LocationInfo>(location_extractor),
        extract::<DateInfo>(date_extractor)
    ))
    // 3. 合并提取结果
    .map(|(persons, locations, dates)| {
        DocumentMetadata {
            persons,
            locations,
            dates
        }
    });
```

### 4. 工具调用场景

```rust
let tool_pipeline = pipeline::new()
    // 1. 分析用户意图
    .map(analyze_intent)
    // 2. 条件分支处理不同工具
    .chain(conditional!(Intent,
        Calculate => calculator_tool,
        Search => search_tool,
        Translate => translation_tool
    ))
    // 3. 格式化结果
    .map(format_tool_response);
```

### 5. 错误处理和重试场景

```rust
let robust_pipeline = pipeline::new()
    .map(initial_process)
    .and_then(|input| async move {
        // 带重试的 API 调用
        retry::retry(ExponentialBackoff::default(), || async {
            api_call(input).await
        }).await
    })
    .or_else(|error| async move {
        // 错误处理和降级策略
        fallback_process(error).await
    });
```

### 6. 动态上下文管理

```rust
let context_aware_pipeline = pipeline::new()
    // 1. 并行获取各种上下文
    .chain(parallel!(
        get_user_context,
        get_conversation_history,
        get_relevant_documents
    ))
    // 2. 合并上下文
    .map(|(user, history, docs)| {
        Context::new()
            .with_user(user)
            .with_history(history)
            .with_documents(docs)
    })
    // 3. 使用上下文生成响应
    .prompt(context_aware_model);
```

### 优势

1. **模块化**
- 每个处理步骤都是独立的
- 易于测试和维护
- 支持代码重用

2. **灵活性**
- 可以动态组合不同的处理步骤
- 支持条件分支和并行处理
- 易于添加新功能

3. **可靠性**
- 内置错误处理
- 支持重试机制
- 可以实现降级策略

4. **性能**
- 支持并行处理
- 异步操作优化
- 资源利用效率高

这种 Pipeline 架构让 Agent 能够处理复杂的任务流程，同时保持代码的清晰性和可维护性。


## 5.Extractor


Extractor 是 Rig 框架中用于从非结构化文本中提取结构化数据的组件。让我详细解析：

### 1. Extractor 的作用

```rust
// 1. 定义要提取的结构化数据格式
#[derive(Debug, Deserialize, JsonSchema, Serialize)]
struct Person {
    first_name: Option<String>,
    last_name: Option<String>,
    job: Option<String>,
}

// 2. 创建和使用提取器
let data_extractor = openai_client.extractor::<Person>("gpt-4").build();
let person = data_extractor
    .extract("Hello my name is John Doe! I am a software engineer.")
    .await?;
```

主要功能：
- 将非结构化文本转换为结构化数据
- 自动生成 schema 并指导 LLM 提取
- 处理提取结果的序列化/反序列化

### 2. 实现原理

1. **Extractor 结构**
```rust
pub struct Extractor<T, M: CompletionModel> {
    agent: Agent<M>,
    _t: PhantomData<T>,
}
```

2. **构建过程**
```rust
impl<T: JsonSchema + for<'a> Deserialize<'a> + Serialize + Send + Sync> ExtractorBuilder<T, M> {
    pub fn build(self) -> Extractor<T, M> {
        // 1. 获取类型的 JSON Schema
        let schema = schema_for!(T);
        
        // 2. 构建提示模板
        let preamble = format!(
            "Extract structured data according to this JSON schema:\n{}\n\n\
             Return only the JSON data without any additional text.",
            serde_json::to_string_pretty(&schema).unwrap()
        );
        
        // 3. 创建专门的提取 agent
        let agent = self.agent_builder
            .preamble(&preamble)
            .build();
            
        Extractor {
            agent,
            _t: PhantomData,
        }
    }
}
```

3. **提取过程**
```rust
impl<T: for<'a> Deserialize<'a>, M: CompletionModel> Extractor<T, M> {
    pub async fn extract(&self, text: &str) -> Result<T, ExtractionError> {
        // 1. 发送提取请求给 LLM
        let response = self.agent.prompt(text).await?;
        
        // 2. 解析 JSON 响应
        let result: T = serde_json::from_str(&response)?;
        
        Ok(result)
    }
}
```

### 3. 具体应用场景

1. **个人信息提取**
```rust
#[derive(Deserialize, JsonSchema)]
struct PersonInfo {
    name: Option<String>,
    age: Option<u8>,
    email: Option<String>,
    occupation: Option<String>,
}

// 从简历文本中提取信息
let info = extractor
    .extract("John Doe, 30 years old, john@email.com, Senior Developer")
    .await?;
```

2. **商品信息提取**
```rust
#[derive(Deserialize, JsonSchema)]
struct ProductInfo {
    name: String,
    price: f64,
    description: Option<String>,
    specifications: HashMap<String, String>,
}

// 从产品描述中提取信息
let product = extractor
    .extract("iPhone 15, $999, Latest smartphone with A16 chip...")
    .await?;
```

3. **事件信息提取**
```rust
#[derive(Deserialize, JsonSchema)]
struct EventInfo {
    title: String,
    date: String,
    location: Option<String>,
    participants: Vec<String>,
}

// 从会议记录中提取信息
let event = extractor
    .extract("Team meeting on 2024-03-15 at HQ, with Alice, Bob, and Carol")
    .await?;
```

4. **文档元数据提取**
```rust
#[derive(Deserialize, JsonSchema)]
struct DocumentMetadata {
    title: String,
    authors: Vec<String>,
    publication_date: Option<String>,
    keywords: Vec<String>,
}

// 从学术论文中提取元数据
let metadata = extractor
    .extract("Title: AI in 2024, Authors: Smith, J., Jones, K...")
    .await?;
```

### 4. 优点

1. **类型安全**
- 使用 Rust 的类型系统
- 编译时验证
- 自动生成 schema

2. **灵活性**
- 支持复杂的数据结构
- 可选字段处理
- 嵌套对象支持

3. **易用性**
- 声明式数据结构
- 自动处理序列化
- 清晰的错误处理

4. **可扩展性**
- 支持自定义数据类型
- 可以添加验证逻辑
- 集成其他处理步骤

### 5. 最佳实践

1. **字段定义**
```rust
#[derive(Deserialize, JsonSchema)]
struct Data {
    // 使用 Option 处理可选字段
    optional_field: Option<String>,
    
    // 添加字段文档
    #[schemars(description = "User's age in years")]
    age: u8,
    
    // 使用合适的类型
    timestamp: chrono::DateTime<chrono::Utc>,
}
```

2. **错误处理**
```rust
match extractor.extract(text).await {
    Ok(data) => process_data(data),
    Err(ExtractionError::JsonError(e)) => handle_json_error(e),
    Err(ExtractionError::LLMError(e)) => handle_llm_error(e),
    Err(e) => handle_other_error(e),
}
```

Extractor 是一个强大的工具，特别适合需要从非结构化文本中提取结构化信息的场景。 