---
{"dg-publish":true,"permalink":"/ai/1/langchain/","tags":["gardenEntry"]}
---


## langchain实践内容介绍：


**TextSplitter** ：文本分割，将一个文本分割成更小的块，以便适合模型适合的**上下文窗口** [^1]
。使用了 `lanchain.text_splitter` 下的 `RecursiveCharacterTextSplitter`

**VectorStore** ：向量存储，提供向量存储和根据用户输入的问题进行相似性检索。
`使用了Chroma`。相似的还有facebook的`Faiss`、 `Milvus`、`Pinecone`等

**EmbeddingsModel** ： 文本嵌入模型，提供将文本编码为向量。使用了Openai的embedding :` text-embedding-ada-002`  该模型在 文本查找、代码查找、句子相似度、文本分类上，评分都较高，所以用该模型


![CleanShot 2024-05-30 at 17.14.10@2x.png| 350](/img/user/ai/1/%E9%99%84%E4%BB%B6/CleanShot%202024-05-30%20at%2017.14.10@2x.png)![CleanShot 2024-05-30 at 17.14.21@2x.png|350](/img/user/ai/1/%E9%99%84%E4%BB%B6/CleanShot%202024-05-30%20at%2017.14.21@2x.png)
![CleanShot 2024-05-30 at 17.13.37@2x.png|350](/img/user/ai/1/%E9%99%84%E4%BB%B6/CleanShot%202024-05-30%20at%2017.13.37@2x.png)![CleanShot 2024-05-30 at 17.13.50@2x.png|350](/img/user/ai/1/%E9%99%84%E4%BB%B6/CleanShot%202024-05-30%20at%2017.13.50@2x.png)

**Retriever** ：使用了Chroma 配套的`similarity_search_with_relevance_scores` [^2]方法实现，同时还提供了其他方法：![[CleanShot 2024-05-30 at 17.23.01@2x.png \| ]]


**memory**：记录历史问题回答，使用了默认的`ConversationBufferMemory` ，保留完整会话
![CleanShot 2024-05-30 at 17.26.02@2x.png](/img/user/ai/1/%E9%99%84%E4%BB%B6/CleanShot%202024-05-30%20at%2017.26.02@2x.png)

**Model** ：使用了 `GPT4` 、`claude`、`百川` 三个模型做了对比

**agent tool**：定义了三个tool ，分别为：`general_func`、`retrival_func`、`search_func`

> general_func : 用大模型回答通用性问题，防止乱编造，如果不知道直接回答`不知道`
> retrival_func ：回答股票相关的本地知识库问题 ， 防止乱编造，如果不知道直接回答`不知道`
> search_func ：用于搜索一些实时性问题，例如 今天的日期、新闻、天气等 

- 搜索用了 google、baidu(不能用)、bing(不能用)、360(效果不是很好)
## 调整策略：
### 1.文本分割调整：
调整`text_spliter`：对chunk_size 和chunk_overlap 调整；
- 1.以[10 , 5] 分割，发现存储了半个句子，导致回答不理想
![CleanShot 2024-05-30 at 19.27.29@2x.png](/img/user/ai/1/%E9%99%84%E4%BB%B6/CleanShot%202024-05-30%20at%2019.27.29@2x.png)
然后以[50，10]发现回答还是不理想；


最后通过统计问答对，取问答总长 `chunk_size=300`作为切分长度，`chunk_overlap = 50`
![CleanShot 2024-05-30 at 20.17.17@2x.png|400](/img/user/ai/1/%E9%99%84%E4%BB%B6/CleanShot%202024-05-30%20at%2020.17.17@2x.png)

### 2. 调整vector trunk_size 
优化转化存储时间，分别调整为10，50，100 
![trunksize=10.png](/img/user/ai/1/%E9%99%84%E4%BB%B6/trunksize=10.png)


![[trunksize=50.png \|trunksize=50.png ]]![[trunksize=100.png]]

### 3. 更改推理策略：
> 原本使用`agent_scratchpad`[^3]，由于其反复思考和处理action导致会重复进行很多次，导致延时很长，因此调整为：不需要模型思考，直接根据用户提供的query 和 tool的description  让模型思考选择使用哪个工具，极大节省了响应时间...**(其实这里感觉有所牺牲模型的反复思考能力...)**
  
### 4. 调整模型
> 采用不同的模型看效果和回答时长

```
openai：
1.共处理耗时： 5.52(总结问题)+2.39(生成工具)+5.32(生成回答) 共耗时
2.回答内容可以
```
![CleanShot 2024-05-30 at 20.44.04@2x.png](/img/user/ai/1/%E9%99%84%E4%BB%B6/CleanShot%202024-05-30%20at%2020.44.04@2x.png)
![CleanShot 2024-05-30 at 20.44.27@2x.png](/img/user/ai/1/%E9%99%84%E4%BB%B6/CleanShot%202024-05-30%20at%2020.44.27@2x.png)


```
claude：
1.耗时相对open较少，时长为：1.95(推理工具选择)+3.85(生成答案)  = 5.8s
2.回答内容总会带入 prompt信息........
```
![CleanShot 2024-05-30 at 20.42.15@2x.png](/img/user/ai/1/%E9%99%84%E4%BB%B6/CleanShot%202024-05-30%20at%2020.42.15@2x.png)
![CleanShot 2024-05-30 at 20.40.50@2x.png](/img/user/ai/1/%E9%99%84%E4%BB%B6/CleanShot%202024-05-30%20at%2020.40.50@2x.png)

```
百川：耗时最短，但回答的问题有时候会回答"不知道"，模型的理解导致的准确度有所欠缺
```
![CleanShot 2024-05-30 at 21.03.32@2x.png](/img/user/ai/1/%E9%99%84%E4%BB%B6/CleanShot%202024-05-30%20at%2021.03.32@2x.png)

![CleanShot 2024-05-30 at 21.02.32@2x.png](/img/user/ai/1/%E9%99%84%E4%BB%B6/CleanShot%202024-05-30%20at%2021.02.32@2x.png)

==总结：百川耗时相对最短，问题回答相对也较准，偶尔会回答"不知道"，其次openai，耗时重但回答效果最好，claude表现均不满意，所以最终选择了百川==

### 5. 优化调整prompt
> 调整prompt内容和description内容 ，使模型更好理解用户问题输出...
![CleanShot 2024-05-30 at 21.14.34@2x.png](/img/user/ai/1/%E9%99%84%E4%BB%B6/CleanShot%202024-05-30%20at%2021.14.34@2x.png)
![CleanShot 2024-05-30 at 21.15.26@2x.png](/img/user/ai/1/%E9%99%84%E4%BB%B6/CleanShot%202024-05-30%20at%2021.15.26@2x.png)


## 展示：
> gradio

![CleanShot 2024-05-30 at 21.37.23@2x.png](/img/user/ai/1/%E9%99%84%E4%BB%B6/CleanShot%202024-05-30%20at%2021.37.23@2x.png)

## 回顾与思考


1.agent_scratchpad 会来回思考用哪个答案 一直到得出一个自己满意的答案才会输出….思考感觉是它的核心，但是太慢了，有没有更好的优化策略？ 

2.大模型发展至今，agent相比factory微调效果较好，对硬件资源也没有太高要求，时间占用也少，大部分在embedding上，请问微调相比agent有哪些独特的优势？

3.实体命名抽取 例如通过用户输入一句话，找到关键词，然后根据关键词搜索数据库...这块该如何实现？  

4.向量数据库如chroma和neo4j有什么本质上的区别？













[^1]: 在进行预测或生成文本时，所考虑的前一个词元（token）或文本片段的大小范围。在语言模型中，上下文窗口对于理解和生成与特定上下文相关的文本至关重要。较大的上下文窗口可以提供更丰富的语义信息、消除歧义、处理上下文依赖性，并帮助模型生成连贯、准确的文本，还能更好地捕捉语言的上下文相关性，使得模型能够根据前文来做出更准确的预测或生成。

[^2]:根据用户问题和存储的问题做相似度计算 并打分，按照打分提取前五名数据，

[^3]:`agent_scratchpad` 是我们添加代理 (Agents) 已经执行的 _每个_ 思考或动作的地方。所有的思考和动作（在 _当前_ 代理 (Agents) 执行器链中）都可以被 _下一个_ 思考-动作-观察循环访问，从而实现代理 (Agents) 动作的连续性。