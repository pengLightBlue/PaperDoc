# 一文彻底搞懂大模型 - RAG（检索、增强、生成）

![RAG](https://api.ibos.cn/v4/weapparticle/accesswximg?aid=88690&url=aHR0cHM6Ly9tbWJpei5xcGljLmNuL3N6X21tYml6X3BuZy83VFdSaGg0eGlja2trRUs4WVhjQ2lhUUlIMWRBandranlNb2ZudmJlUTR3UzJ5N29Cc1gxdEVxbGdHS3NEamljYmRHRjVHaWNuTG9WTnBzRmtWZjBHczBvRVEvNjQwP3d4X2ZtdD1wbmcmYW1w;from=appmsg)

**RAG（Retrieval-Augmented Generation，检索增强生成） 是一种结合了信息检索技术与语言生成模型的人工智能技术。*该技术通过从外部知识库中检索相关信息，并将其作为提示（Prompt）输入给大型语言模型（LLMs），以增强模型处理知识密集型任务的能力，如问答、文本摘要、内容生成等。RAG模型由Facebook AI Research（FAIR）团队于2020年首次提出，并迅速成为大模型应用中的热门方案。

![](https://api.ibos.cn/v4/weapparticle/accesswximg?aid=88690&url=aHR0cHM6Ly9tbWJpei5xcGljLmNuL3N6X21tYml6X3BuZy83VFdSaGg0eGlja2trRUs4WVhjQ2lhUUlIMWRBandranlNUlVuQ1pPdXB4eklpYWVyMEJSZ0M2TmljTzZPdll6M0dRTUJrWFBBQnRmb1ZNcFFPRTNHSGRpYUVRLzY0MD93eF9mbXQ9cG5nJmFtcA==;from=appmsg)

## 一、检索增强生成（RAG）

**什么是**RAG？RAG（Retrieval-Augmented Generation，检索增强生成），RAG**是一种 AI 框架，它将传统信息检索系统（例如数据库）的优势与生成式大语言模型 (LLM) 的功能结合在一起。**

LLM通过将这些额外的知识与自己的语言技能相结合，可以撰写更准确、更具时效性且更贴合具体需求的文字。

![什么是RAG](https://api.ibos.cn/v4/weapparticle/accesswximg?aid=88690&url=aHR0cHM6Ly9tbWJpei5xcGljLmNuL3N6X21tYml6X3BuZy83VFdSaGg0eGlja2trRUs4WVhjQ2lhUUlIMWRBandranlNdmw3OHhRd2NBVjRPSXlrdldSUXhUdFVJNWhIaFFYZFk0eDJDdnYxeVhpYk1yZ3JsVTAzOUFFdy82NDA/d3hfZm10PXBuZyZhbXA=;from=appmsg)

如何理解RAG？通过上一个问题，我们知道了什么是RAG？了解到RAG是一种结合了信息检索、文本增强和文本生成的自然语言处理（NLP）的技术。

RAG的目的是通过从外部知识库检索相关信息来辅助大语言模型生成更准确、更丰富的文本内容。那我们如何理解RAG的检索、增强和生成呢？

1.  **检索**：检索是RAG流程的第一步，从预先建立的知识库中检索与问题相关的信息。这一步的目的是为后续的生成过程提供有用的上下文信息和知识支撑。

2.  **增强**：RAG中增强是将检索到的信息用作生成模型（即大语言模型）的上下文输入，以增强模型对特定问题的理解和回答能力。这一步的目的是将外部知识融入生成过程中，使生成的文本内容更加丰富、准确和符合用户需求。**通过增强步骤，LLM模型能够充分利用外部知识库中的信息。**

3.  生成：生成是RAG流程的最后一步。这一步的目的是结合LLM生成符合用户需求的回答。生成器会利用检索到的信息作为上下文输入，并结合大语言模型来生成文本内容。

RAG的"检索、增强、生成"，谁增强了谁，谁生成了答案，主语很重要。是从知识库中检索到的问答对，增强了LLM的提示词（prompt），LLM拿着增强后的Prompt生成了问题答案。

![如何理解RAG](https://api.ibos.cn/v4/weapparticle/accesswximg?aid=88690&url=aHR0cHM6Ly9tbWJpei5xcGljLmNuL3N6X21tYml6X3BuZy83VFdSaGg0eGlja2trRUs4WVhjQ2lhUUlIMWRBandranlNSEkxTHd3TmJMd2VZakN2RlltaWI0OGNrM3FoaDN1bkRxSnVGN3g5aWJkQjVyOTNMWWRCU2xPcEEvNjQwP3d4X2ZtdD1wbmcmYW1w;from=appmsg)

**如何使用RAG？******了解了什么是RGA，同步也理解了RAG的检索、增强和生成。那我们如何使用RAG呢？接下来以RAG搭建知识问答系统具体步骤为例，来讲解如何使用RAG？****

1.  **数据准备与知识库构建**：

-   **收集数据：**首先，需要收集与问答系统相关的各种数据，这些数据可以来自文档、网页、数据库等多种来源。

-   **数据清洗：**对收集到的数据进行清洗，去除噪声、重复项和无关信息，确保数据的质量和准确性。

-   **知识库构建：**将清洗后的数据构建成知识库。这通常包括将文本分割成较小的片段（chunks），使用文本嵌入模型（如GLM）将这些片段转换成向量，并将这些向量存储在向量数据库（如FAISS、Milvus等）中。

-   **检索模块设计：**

-   **问题向量化：**当用户输入查询问题时，使用相同的文本嵌入模型将问题转换成向量。

-   **相似度检索：**在向量数据库中检索与问题向量最相似的知识库片段（chunks）。这通常通过计算向量之间的相似度（如余弦相似度）来实现。

-   **结果排序：**根据相似度得分对检索到的结果进行排序，选择最相关的片段作为后续生成的输入。

-   生成模块设计：

-   **上下文融合**：将检索到的相关片段与原始问题合并，形成更丰富的上下文信息。

-   **大语言模型生成**：使用大语言模型（如GLM）基于上述上下文信息生成回答。大语言模型会学习如何根据检索到的信息来生成准确、有用的回答。

******大家可以结合自己的业务领域知识，开始搭建医疗、法律、产品知识问答。先搭建Demo，然后工作中不断完善知识库问答对。******

![如何使用RAG](https://api.ibos.cn/v4/weapparticle/accesswximg?aid=88690&url=aHR0cHM6Ly9tbWJpei5xcGljLmNuL3N6X21tYml6X3BuZy83VFdSaGg0eGlja2trRUs4WVhjQ2lhUUlIMWRBandranlNem1RWkNsU0hLOUFSUXZpYmtXMW80dnI2am1Yd3E1M01VcFk3QW5oRTg2ZWZUZTBjYW50enM1dy82NDA/d3hfZm10PXBuZyZhbXA=;from=appmsg)

## 二、RAG的原理、流程及架构

**RAG工作原理是什么？大型语言模型（LLM）面临两个问题，第一个问题是LLM会产生幻觉，第二个是LLM的知识中断。**

1.  知识截止：当 LLM 返回的信息与模型的训练数据相比过时时。每个基础模型都有知识截止，这意味着其知识仅限于训练时可用的数据。

2.  幻觉：当模型自信地做出错误反应时，就会发生幻觉。

**检索增强生成 (RAG) 摆脱了知识限制，整合了外部数据，从外部知识库中检索相关信息，增强模型的生成能力。**

![RAG工作原理是什么](https://api.ibos.cn/v4/weapparticle/accesswximg?aid=88690&url=aHR0cHM6Ly9tbWJpei5xcGljLmNuL3N6X21tYml6X3BuZy83VFdSaGg0eGlja2trRUs4WVhjQ2lhUUlIMWRBandranlNaWFpYVdsa09wN0puaWNrSVpNZExpY1NYankyZmljYkRjU2t5TFZMTUgyajRiTGN0REpGZWZ1alRZbUEvNjQwP3d4X2ZtdD1wbmcmYW1w;from=appmsg)

**RAG工作流程是什么？**通过检索增强技术，将用户查询与索引知识融合，利用大语言模型生成准确回答。

1.  知识准备：收集并转换知识文档为文本数据，进行预处理和索引。

2.  嵌入与索引：使用嵌入模型将文本转换为向量，并存储在向量数据库中。

3.  查询检索：用户查询转换为向量，从数据库中检索相关知识。

4.  提示增强：结合检索结果构建增强提示模版。

5.  生成回答：大语言模型根据增强模版生成准确回答。

![RAG工作流程是什么](https://api.ibos.cn/v4/weapparticle/accesswximg?aid=88690&url=aHR0cHM6Ly9tbWJpei5xcGljLmNuL3N6X21tYml6X2pwZy83VFdSaGg0eGlja2trRUs4WVhjQ2lhUUlIMWRBandranlNa3lIZU9WU1ZkdnZiUDNvYmpuNWNwVHBINFJMaWEzMEZ2V044N3BKOXBBaWNpYW0yUUJqYnpyR21BLzY0MD93eF9mbXQ9anBlZw==)

RAG技术架构是什么？RAG技术架构主要由两个核心模块组成，检索模块（Retriever）和生成模块（Generator）。

1.  **检索模块（Retriever）：**

-   文本嵌入：使用预训练的文本嵌入模型（如GLM）将查询和文档转换成向量表示，以便在向量空间中进行相似度计算。

-   向量搜索：利用高效的向量搜索技术（如FAISS、Milvus等向量数据库）在向量空间中检索与查询向量最相似的文档或段落。

-   双塔模型：检索模块常采用双塔模型（Dual-Encoder）进行高效的向量化检索。双塔模型由两个独立的编码器组成，一个用于编码查询，另一个用于编码文档。这两个编码器将查询和文档映射到相同的向量空间中，以便进行相似度计算。

-   **生成模块（Generator）：**

-   强大的生成模型：生成模块通常使用在大规模数据上预训练的生成模型（如GLM），这些模型在生成自然语言文本方面表现出色。

-   上下文融合：生成模块将检索到的相关文档与原始查询合并，形成更丰富的上下文信息，作为生成模型的输入。

-   生成过程：生成模型根据输入的上下文信息，生成连贯、准确且信息丰富的回答或文本。

**结合高效的检索模块（**Retriever**）与强大的生成模型（**Generator**），实现基于外部知识增强的自然语言生成能力。**

![RAG技术架构是什么？](https://api.ibos.cn/v4/weapparticle/accesswximg?aid=88690&url=aHR0cHM6Ly9tbWJpei5xcGljLmNuL3N6X21tYml6X3BuZy83VFdSaGg0eGlja2trRUs4WVhjQ2lhUUlIMWRBandranlNOU9kUkM1aWFtbTZZYWljOU1mMDRhekhhSFBWM2pYbUVsSEtld0MxSzhOZWNtTjVzWXVxaWNnN2h3LzY0MD93eF9mbXQ9cG5nJmFtcA==;from=appmsg)

