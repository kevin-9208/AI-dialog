# 《Attention Is All You Need》论文深度解读

2017年，来自 Google Brain 和 Google Research 的研究人员发表了论文：

Attention Is All You Need

作者包括：

* Ashish Vaswani
* Noam Shazeer
* Niki Parmar
* Jakob Uszkoreit
* Llion Jones
* Illia Polosukhin

这篇论文的重要性，相当于：

> 深度学习领域从「蒸汽机时代」进入「电气时代」。

今天的：

* OpenAI GPT系列
* Anthropic Claude系列
* Google DeepMind Gemini系列
* Meta Llama系列
* DeepSeek DeepSeek系列

本质上全部建立在 Transformer 架构之上。

---

# 一、Transformer出现前的世界

在2017年之前：

自然语言处理主要依赖：

## 1 RNN

Recurrent Neural Network

结构：

```text
我 → 爱 → 你
↓   ↓   ↓
h1→ h2→ h3
```

特点：

* 顺序处理
* 前一个词影响后一个词

问题：

计算无法并行

例如：

```text
h3
必须等待
h2

h2
必须等待
h1
```

GPU效率极低。

---

## 2 LSTM

Long Short-Term Memory

解决：

* 长距离依赖
* 梯度消失

例如：

```text
小明出生在北京。

......

几十个词以后

他现在住在哪里？
```

LSTM比RNN记忆更长。

但是：

本质还是串行。

速度依旧慢。

---

## 3 Seq2Seq + Attention

机器翻译时代经典结构：

```text
Encoder
↓
Context
↓
Decoder
```

例如：

```text
I love you
↓

我 爱 你
```

研究人员发现：

Attention效果特别好。

于是有人提出：

> 既然Attention这么厉害，
> 为什么还要RNN？

于是诞生了Transformer。

---

# 二、论文最核心思想

论文标题：

```text
Attention Is All You Need

注意力就是你需要的一切
```

意思是：

```text
去掉：

RNN
LSTM
CNN

只保留：

Attention
```

整个模型：

```text
全靠注意力机制工作
```

这在当时非常激进。

很多人觉得不可能。

结果后来改变了整个AI行业。

---

# 三、Attention是什么

举例：

```text
小明把书借给了小红，
因为她需要复习考试。
```

问：

```text
她
指谁？
```

人类知道：

```text
她 = 小红
```

因为：

```text
需要复习考试
↓
更像小红
```

这就是注意力。

模型会：

```text
关注
最相关的词
```

而不是机械顺序读取。

---

# 四、Self-Attention自注意力

Transformer核心。

例如句子：

```text
I love machine learning
```

每个词：

```text
I
love
machine
learning
```

都要查看：

```text
其它所有词
```

形成关系网络。

```text
I -------- learning
 \        /
  \      /
   love
    |
 machine
```

---

## Self-Attention计算过程

输入：

```text
X
```

先变换：

```text
Q Query
K Key
V Value
```

得到：

```text
Q = XWq

K = XWk

V = XWv
```

---

### Query

查询：

```text
我在找什么
```

---

### Key

标签：

```text
我是什么
```

---

### Value

内容：

```text
我携带的信息
```

---

## 举例

词：

```text
Apple
```

Query：

```text
我想找水果
```

Key：

```text
我是水果
```

匹配成功。

Attention增大。

---

# 五、Attention公式

Transformer最著名公式：

Attention(Q,K,V)=softmax\left(\frac{QK^T}{\sqrt{d_k}}\right)V

看起来吓人。

实际上只分三步。

---

## 第一步

计算相关性：

QK^T

得到：

```text
词与词之间
关联程度
```

例如：

```text
小明 → 小红

0.8
```

```text
小明 → 考试

0.1
```

---

## 第二步

除以：

\sqrt{d_k}

作用：

防止数值过大。

称为：

Scaled Dot Product。

---

## 第三步

Softmax

把分数变概率：

```text
0.8
0.1
0.1
```

变：

```text
80%
10%
10%
```

---

最后：

```text
概率 × V
```

得到最终输出。

---

# 六、多头注意力 Multi-Head Attention

论文另一大创新。

如果只有一个Attention：

```text
只能看一种关系
```

例如：

```text
主谓关系
```

但语言关系很多：

```text
语法
语义
时态
代词
逻辑
```

于是：

Transformer同时使用多个Attention。

```text
Head1
Head2
Head3
...
Head8
```

论文中：

```text
8个头
```

结构：

```text
输入
 ↓

Head1
Head2
Head3
...
Head8

 ↓

Concat

 ↓

Linear
```

---

理解成：

```text
8个专家同时分析一句话
```

最终投票。

---

# 七、位置编码 Positional Encoding

Attention有个问题：

```text
不知道顺序
```

例如：

```text
猫咬狗
```

和

```text
狗咬猫
```

词一样。

Attention无法区分。

---

因此加入：

Position Encoding。

论文使用：

```text
sin
cos
```

函数。

形式：

PE(pos,2i)=\sin\left(pos/10000^{2i/d_{model}}\right)

以及：

PE(pos,2i+1)=\cos\left(pos/10000^{2i/d_{model}}\right)

---

作用：

```text
第1个词
第2个词
第3个词
```

拥有不同坐标。

模型就知道顺序。

---

# 八、Transformer整体架构

经典结构：

```text
Encoder × 6
↓
Decoder × 6
```

论文原版：

```text
6层Encoder
6层Decoder
```

---

## Encoder

结构：

```text
Input
↓
Multi Head Attention
↓
Add & Norm
↓
Feed Forward
↓
Add & Norm
```

---

### Feed Forward

实际上是：

```text
Linear
↓
ReLU
↓
Linear
```

公式：

FFN(x)=\max(0,xW_1+b_1)W_2+b_2

作用：

增强表达能力。

---

## Residual

残差连接：

```text
输出 = 输入 + 新结果
```

思想来自：

ResNet

好处：

* 防梯度消失
* 深层训练稳定

---

## LayerNorm

归一化：

```text
均值=0
方差=1
```

训练更稳定。

---

# 九、Decoder

Decoder多一个模块：

```text
Masked Self Attention
```

---

原因：

生成文本时：

```text
不能偷看未来
```

例如：

```text
我 爱 ?
```

预测：

```text
你
```

时不能先看到：

```text
你
```

否则作弊。

---

Mask：

```text
未来位置
全部遮挡
```

---

# 十、为什么Transformer这么强

## 1 完全并行

RNN：

```text
一个一个处理
```

Transformer：

```text
一次处理整个句子
```

GPU利用率暴涨。

---

## 2 长距离依赖

RNN：

```text
词1 → 词100
```

路径很长。

Transformer：

```text
直接连接
```

```text
词1 ↔ 词100
```

一步到达。

---

## 3 可扩展

参数：

```text
1亿
10亿
100亿
1000亿
```

都能扩展。

于是出现：

* GPT
* Claude
* Gemini
* Llama

等超大模型。

---

# 十一、GPT与Transformer关系

很多人误以为：

```text
GPT = Transformer
```

实际上：

```text
GPT ⊂ Transformer
```

GPT只是Transformer变种。

---

GPT使用：

```text
Decoder Only
```

结构：

```text
Decoder
Decoder
Decoder
...
```

没有Encoder。

---

例如：

OpenAI GPT-1

```text
12层
117M参数
```

后来：

GPT-2

```text
1.5B
```

GPT-3

```text
175B
```

开启大模型时代。

---

# 十二、BERT与Transformer关系

BERT 使用：

```text
Encoder Only
```

结构：

```text
Encoder
Encoder
Encoder
...
```

适合：

* 分类
* 搜索
* 语义匹配
* 信息抽取

---

形成三大流派：

| 架构              | 代表   |
| --------------- | ---- |
| Encoder Only    | BERT |
| Decoder Only    | GPT  |
| Encoder Decoder | T5   |

---

# 十三、Transformer带来的革命

2017以前：

```text
RNN时代
```

2017以后：

```text
Transformer时代
```

随后诞生：

* BERT（2018）
* GPT（2018）
* T5（2019）
* ViT（2020）
* ChatGPT（2022）

再到今天的大模型浪潮。

---

# 一张图理解Transformer

```text
输入句子
     ↓
词向量Embedding
     ↓
位置编码Positional Encoding
     ↓
┌─────────────────┐
│ Multi Head Attn │
└─────────────────┘
     ↓
 Add & Norm
     ↓
 Feed Forward
     ↓
 Add & Norm
     ↓
（重复N层）
     ↓
 Encoder输出
     ↓
 Decoder
     ↓
 Linear
     ↓
 Softmax
     ↓
 下一个Token
```

# 一句话总结

Transformer的本质可以概括为：

> **利用 Self-Attention 让每个 Token 与所有 Token 建立联系，再通过多头注意力、位置编码、残差连接和前馈网络进行高效并行计算，从而取代 RNN/LSTM，成为现代大语言模型和生成式 AI 的基础架构。**

如果继续深入，下一个层次就是：

1. Token → Embedding 的数学细节；
2. Self-Attention 的矩阵运算全过程（带数字实例）；
3. GPT、BERT、T5、Llama、DeepSeek 的架构差异；
4. KV Cache、RoPE、MoE、FlashAttention 等现代 Transformer 演化路线。
