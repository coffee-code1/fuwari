---
title: 大模型应用开发
published: 2026-08-03
updated: 2026-08-03
description: 初步认识AI以及大模型
image: ''
tags: [大模型应用,NLP,LLM]
category: 大模型应用
draft: false 
---
# 一、初步认识AI与大模型

## 1.1 AI的含义
AI:人工智能（Artificial Intelligence），使机器能够像人类一样思考、学习和解决问题的技术。
## 1.2 AI的发展
从**规则和符号AI**的时代到**机器学习**再到现在的**深度学习**<br>
| 阶段 | 时间区间 | 核心特点 | 典型应用 |
| ---- | -------- | -------- | -------- |
| 规则和符号AI的时代 | 1950s–1980s | 基于逻辑和规则，使用符号表示知识和推理。依赖预定义的知识库和推理规则 | 医学诊断、化学结构分析 |
| 机器学习 | 1980s–2010s | 基于数据，通过统计和优化方法训练模型。包括监督学习、无监督学习和强化学习等子领域 | 游戏AI、推荐引擎 |
| 深度学习 | 2010s至今 | 模仿人脑的结构和功能，使用多层神经元网络处理复杂任务，例如卷积神经网络 | 图像识别、自然语言处理 |
>[!TIP]
现在的GPT  ds等**大模型（LLM）**都是基于深度学习里面**自然语言处理（NLP）**的应用中的**transformer**实现的
## 1.3大模型底层原理
- T：基于Transformer的神经网络
- P：通过大量数据预训练，掌握自然语言规律
- G：基于上文计算概率，生成下一个token

# 二、大模型应用开发
## 2.1 大模型部署
|部署方式|优点|缺点|
| ---- | ---- | ---- |
|**云部署**|前期成本低<br>部署维护简单<br>弹性扩展<br>全球访问|数据隐私<br>网络依赖<br>长期成本高|
|**开放API**|前期成本极低<br>无需部署<br>无需维护<br>全球访问|数据隐私<br>网络依赖<br>长期成本高<br>定制限制|
|**本地部署**|数据安全<br>不依赖外部网络<br>长期成本低<br>高度定制|初始成本高<br>维护复杂<br>部署周期长|

### 2.1.1 云部署
国内知名的云服务平台都提供了全球知名的大模型的私有部署功能，甚至还提供了这些模型的API开放平台，无需部署就能访问。

| 云平台 | 公司 | 地址 |
| ---- | ---- | ---- |
| 阿里百炼 | 阿里巴巴 | https://bailian.console.aliyun.com |
| 腾讯TI平台 | 腾讯 | https://cloud.tencent.com/product/ti |
| 千帆平台 | 百度 | https://console.bce.baidu.com/qianfan/overview |
| SiliconCloud | 硅基流动 | https://siliconflow.cn/zh-cn/siliconcloud |
| 火山方舟-火山引擎 | 字节跳动 | https://www.volcengine.com/product/ark |

### 2.1.2 本地部署
本地部署最简单的一种方案就是使用**ollama**，官网地址：https://ollama.com
>[!NOTE]
其使用的语法跟docker类似

## 2.2 大模型调用
python示例：
~~~python
from openai import OpenAI

# 1.初始化OpenAI客户端
client = OpenAI(api_key="<DeepSeek API Key>", base_url="https://api.deepseek.com")

# 2.发送http请求到大模型
response = client.chat.completions.create(
    model="deepseek-r1",
    messages=[
        {"role": "system", "content": "你是一个热心的AI助手，你的名字叫小团团"},
        {"role": "user", "content": "你好，你是谁？"},
    ],
    stream=False
)

# 3.打印返回结果
print(response.choices[0].message.content)
~~~
>[!NOTE]
云部署以及开放的API接口都需要base_url，然后model名字要准确，本地就是本机地址，还需要api_key，本地不需要。  messages里本质是json字段。

## 2.3 大模型应用是什么
大模型应用是基于大模型的**推理、分析、生成能力**，**结合传统编程能力**，开发出的各种应用<br>
对话应用：
| 大模型 | 对话产品 | 公司 | 地址 |
| ---- | ---- | ---- | ---- |
| GPT-3.5、GPT-4o | ChatGPT | OpenAI | https://chatgpt.com/ |
| Claude 3.5 | Claude AI | Anthropic | https://claude.ai/chats |
| DeepSeek-R1 | DeepSeek | DeepSeek | https://www.deepseek.com/ |
| 文心大模型3.5 | 文心一言 | 百度 | https://yiyan.baidu.com/ |
| 星火3.5 | 讯飞星火 | 科大讯飞 | https://xinghuo.xfyun.cn/desk |
| Qwen-Max | 通义千问 | 阿里巴巴 | https://tongyi.aliyun.com/qianwen/ |
| Moonshoot | Kimi | 月之暗面 | https://kimi.moonshot.cn/ |
| Yi-Large | 零一万物 | 零一万物 | https://platform.lingyiwanwu.com/ |

### 大模型应用
- **文本分析**：数据提取和格式化、坐席质检、舆情分析、文本摘要、知识库
- **多模态**：图片识别、音频识别、视频识别、音频生成、图片生成、视频生成等
- **机器人应用**：AI智能客服机器人开发、对话管理、情感分析、个性化回复等
- **智能体**：AI金融分析、自动化办公、智慧医疗、工业/制造智能体、运维智能体
- **自动驾驶**：计算机视觉处理，车辆自动驾驶

## 2.4 大模型应用开发框架
### 1. P：纯Prompt问答（Pure Prompt）
利用大模型原生推理能力，**直接通过编写Prompt提示词**向大模型提问，完成业务需求。
**优点**：实现最简单，无需额外组件；局限：无法使用私有新知识，复杂场景能力有限。

### 2. A：Agent + Function Calling
**AI智能体自主拆解**复杂任务，并且能够调用外部业务接口、工具函数，联动外部系统完成多步骤复杂业务。
**优点**：适合需要联网查询、操作软件、调用业务API等自动化场景。

### 3. R：RAG 检索增强生成（Retrieval Augmented Generation）
为大模型外挂**私有知识库**；用户提问时先检索知识库相关资料，交给大模型参考，让AI基于自有资料回答。
**优点**：解决大模型知识滞后、不能使用企业内部文档、容易产生幻觉的问题。

### 4. F：Fine-tuning 微调
收集业务专属数据集，在基础大模型之上继续**训练、微调模型参数**，改造模型能力适配垂直行业特有场景。
**特点**：**成本较高**，一般用于需要深度改变模型输出风格、专业领域能力的场景。