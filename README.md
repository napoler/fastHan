# fastHan
## 简介
fastHan是基于[fastNLP](https://github.com/fastnlp/fastNLP)与pytorch实现的中文自然语言处理工具，像spacy一样调用方便。

其内核为基于BERT的联合模型，其在13个语料库中进行训练，可处理中文分词、词性标注、依存分析、命名实体识别四项任务。fastHan共有base与large两个版本，分别利用BERT的前四层与前八层。base版本在总参数量150MB的情况下各项任务均有不错表现，large版本则接近甚至超越SOTA模型。


## 安装指南
fastHan需要以下依赖的包：

torch>=1.0.0

fastNLP>=0.5.0

**版本更新:**
- 1.1版本的fastHan与0.5.5版本的fastNLP会导致importerror。如果使用1.1版本的fastHan，请使用0.5.0版本的fastNLP。
- 1.2版本的fastHan修复了fastNLP版本兼容问题。小于等于1.2版本的fastHan在输入句子的首尾包含**空格、换行**符时会产生BUG。如果字符串首尾包含上述字符，请使用strip函数处理输入字符串。
- 1.3版本的fastHan自动对输入字符串做strip函数处理。

可执行如下命令完成安装：

```
pip install fastHan==1.3
```

## 使用教程
fastHan的使用极为简单，只需两步：加载模型、将句子输入模型。
#### 加载模型
执行以下代码可以加载模型：

```
from fastHan import FastHan
model=FastHan()
```
此时若用户为首次初始化模型，将自动从服务器中下载参数。

模型默认初始化为base，如果使用large版本，可在初始化时加入如下参数：

```
model=FastHan(model_type="large")
```
#### 输入句子
模型对句子进行依存分析、命名实体识别的简单例子如下：

```
sentence="郭靖是金庸笔下的一名男主。"
answer=model(sentence,target="Parsing")
print(answer)
answer=model(sentence,target="NER")
print(answer)
```
模型将会输出如下信息：

```
[[['郭靖', 2, 'top', 'NR'], ['是', 0, 'root', 'VC'], ['金庸', 4, 'nn', 'NR'], ['笔', 5, 'lobj', 'NN'], ['下', 10, 'assmod', 'LC'], ['的', 5, 'assm', 'DEG'], ['一', 8, 'nummod', 'CD'], ['名', 10, 'clf', 'M'], ['男', 10, 'amod', 'JJ'], ['主', 2, 'attr', 'NN'], ['。', 2, 'punct', 'PU']]]
[[['郭靖', 'NR'], ['金庸', 'NR']]]
```
**任务选择**

target参数可在'Parsing'、'CWS'、'POS'、'NER'四个选项中取值，模型将分别进行依存分析、分词、词性标注、命名实体识别任务,模型默认进行CWS任务。其中词性标注任务包含了分词的信息，而依存分析任务又包含了词性标注任务的信息。命名实体识别任务相较其他任务独立。

如果分别运行CWS、POS、Parsing任务，模型输出的分词结果等可能存在冲突。如果想获得不冲突的各类信息，请直接运行包含全部所需信息的那项任务。

模型的POS、Parsing任务均使用CTB标签集。NER使用msra标签集。

**分词风格**

分词风格，指的是训练模型中文分词模块的10个语料库，模型可以区分这10个语料库，设置分词style为S即令模型认为现在正在处理S语料库的分词。所以分词style实际上是与语料库的覆盖面、分词粒度相关的。如本模型默认的CTB语料库分词粒度较细。如果想切换不同的粒度，可以使用模型的set_cws_style函数，例子如下：

>
```
sentence="一个苹果。"
print(model(sentence,'CWS'))
model.set_cws_style('cnc')
print(model(sentence,'CWS'))
```
模型将输出如下内容：

```
[['一', '个', '苹果', '。']]
[['一个', '苹果', '。']]
```
对语料库的选取参考了下方CWS SOTA模型的论文，共包括：SIGHAN 2005的 MSR、PKU、AS、CITYU 语料库，由山西大学发布的 SXU 语料库，由斯坦福的CoreNLP 发布的 CTB6 语料库，由国家语委公布的 CNC 语料库，由王威廉先生公开的微博树库 WTB，由张梅山先生公开的诛仙语料库 ZX，Universal Dependencies 项目的 UD 语料库。

**输入与输出**

输入模型的可以是单独的字符串，也可是由字符串组成的列表。如果输入的是列表，模型将一次性处理所有输入的字符串，所以请自行控制 batch size。

模型的输出是在fastHan模块中定义的sentence与token类。模型将输出一个由sentence组成的列表，而每个sentence又由token组成。每个token本身代表一个被分好的词，有pos、head、head_label、ner四项属性，代表了该词的词性、依存关系、命名实体识别信息。

一则输入输出的例子如下所示：

```
sentence=["我爱踢足球。","林丹是冠军"]
answer=model(sentence,'Parsing')
for i,sentence in enumerate(answer):
    print(i)
    for token in sentence:
        print(token,token.pos,token.head,token.head_label)
```
模型将输入如下内容：

```
0
我 PN 2 nsubj
爱 VV 0 root
踢 VV 2 ccomp
足球 NN 3 dobj
。 PU 2 punct
1
林丹 NR 2 top
是 VC 0 root
冠军 NN 2 attr
！ PU 2 punct
```
可在分词风格中选择'as'、'cityu'进行繁体字分词，这两项为繁体语料库。

此外，由于各项任务共享词表、词嵌入，即使不切换模型的分词风格，模型对繁体字、英文字母、数字均具有一定识别能力。

**切换设备**

可使用模型的set_device函数，令模型在cuda上运行或切换回cpu，示例如下：

```
model.set_device('cuda:0')
model.set_device('cpu')
```
## 模型表现

### 准确率测试
模型在以下数据集进行训练和准确性测试：

- CWS：AS, CITYU, CNC, CTB, MSR, PKU, SXU, UDC, WTB, ZX
- NER：MSRA、OntoNotes
- POS & Parsing：CTB9

注：模型在训练NER OntoNotes时将其标签集转换为与MSRA一致。

模型在ctb分词语料库的前800句进行了速度测试，平均每句有45.2个字符。测试环境为私人电脑， Intel Core i5-9400f + NVIDIA GeForce GTX 1660ti，batch size取16。经测试各项任务运行速度大致相同。

最终模型取得的表现如下：


任务 | CWS | Parsing | POS | NER MSRA | NER OntoNotes | 速度(句/s),cpu|速度(句/s)，gpu
---|---|--- |--- |--- |--- |---|---
SOTA模型 | 97.1 | 85.66,81.71 | 93.15 | 95.25 | 79.92 |——|——
base模型 | 97.27 | 81.22,76.71 | 94.88 | 94.33 | 82.86 |28.8|72.3
large模型 | 97.41 | 85.52,81.38 | 95.66 | 95.50 | 83.82|12.5|64.1 

注：模型在句首加入语料库标签来区分输入句子的任务及语料库，最初测试计算F值时将语料库标签也算入在内，导致CWS、POS分值偏高。现在已修复此处错误并更新了表格中的CWS及POS项。评分略有下降（CWS平均下降0.11，POS平均下降0.29），但仍超越SOTA模型。

表格中单位为百分数。CWS的成绩是10项任务的平均成绩。Parsing中的两个成绩分别代表Fudep和Fldep。SOTA模型的数据来自笔者对网上资料及论文的查阅，如有缺漏请指正，不胜感激。这五项SOTA表现分别来自如下五篇论文：

1. Huang W, Cheng X, Chen K, et al. Toward Fast and Accurate Neural Chinese Word Segmentation with Multi-Criteria Learning.[J]. arXiv: Computation and Language, 2019.
2. Hang Yan, Xipeng Qiu, and Xuanjing Huang. "A Graph-based Model for Joint Chinese Word Segmentation and Dependency Parsing." Transactions of the Association for Computational Linguistics 8 (2020): 78-92.
3. Meng Y, Wu W, Wang F, et al. Glyce: Glyph-vectors for Chinese Character Representations[J]. arXiv: Computation and Language, 2019.
4. Diao S, Bai J, Song Y, et al. ZEN: Pre-training Chinese Text Encoder Enhanced by N-gram Representations[J]. arXiv: Computation and Language, 2019.
5. Jie Z, Lu W. Dependency-Guided LSTM-CRF for Named Entity Recognition[C]. international joint conference on natural language processing, 2019: 3860-3870.


目前此工作准备投稿，如被录用将会把链接附于此处，届时可在论文中得到更多关于模型结构及训练的详细信息。