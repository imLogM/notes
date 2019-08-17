# 手撕 TextRank（01）大佬是怎么实现 Python 版的

作者：LogM

本文原载于 [https://segmentfault.com/u/logm/articles](https://segmentfault.com/u/logm/articles) ，不允许转载~

## 1. 源码来源

TextRank4ZH 源码：[https://github.com/letiantian/TextRank4ZH.git](https://github.com/letiantian/TextRank4ZH.git)

本文对应的源码版本：committed on 3 Jul 2018, `fb1339620818a0b0c16f5613ebf54153faa41636`

TextRank 论文地址：[https://www.aclweb.org/anthology/W04-3252](https://www.aclweb.org/anthology/W04-3252)

## 2. 概述

letiantian 大佬的这个版本，应该是所有 TextRank 的 Python 版本中被点赞最多的。代码写的也非常的简单易懂。

## 3. 开撕

example 文件夹下的程序展示了怎么使用这个版本的 TextRank。有关键词、关键短语、关键句抽取三种功能，我们这边只关注关键句的抽取。

应该很容易看懂吧，先实例化 `TextRank4Sentence`，然后使用 `analyze` 抽取。

```python
# 文件：example/example1.py
# 行数：28
tr4s = TextRank4Sentence()
tr4s.analyze(text=text, lower=True, source = 'all_filters') # 这句是重点

print()
print( '摘要：' )
for item in tr4s.get_key_sentences(num=3):
    print(item.index, item.weight, item.sentence)
```

然后，我们知道重点函数是 `analyze`，我们再来看它是怎么工作的。

```python
# 文件：textrank4zh/TextRank4Sentence.py
# 行数：43
def analyze(self, text, lower = False, 
              source = 'no_stop_words', 
              sim_func = util.get_similarity,
              pagerank_config = {'alpha': 0.85,}):
        """
        Keyword arguments:
        text                 --  文本内容，字符串。
        lower                --  是否将文本转换为小写。默认为False。
        source               --  选择使用words_no_filter, words_no_stop_words, words_all_filters中的哪一个来生成句子之间的相似度。
                                 默认值为`'all_filters'`，可选值为`'no_filter', 'no_stop_words', 'all_filters'`。
        sim_func             --  指定计算句子相似度的函数。
        """
        
        self.key_sentences = []
        
        result = self.seg.segment(text=text, lower=lower)
        self.sentences = result.sentences
        self.words_no_filter = result.words_no_filter
        self.words_no_stop_words = result.words_no_stop_words
        self.words_all_filters   = result.words_all_filters

        options = ['no_filter', 'no_stop_words', 'all_filters']
        if source in options:
            _source = result['words_'+source]
        else:
            _source = result['words_no_stop_words']

        # 这句是重点
        self.key_sentences = util.sort_sentences(
              sentences = self.sentences,
              words     = _source,
              sim_func  = sim_func,
              pagerank_config = pagerank_config)
```

很容易发现，我们需要的内容在 `util.sort_sentences` 这个函数里。

```python
# 文件：textrank4zh/util.py
# 行数：169
def sort_sentences(sentences, words, sim_func = get_similarity, pagerank_config = {'alpha': 0.85,}):
    """将句子按照关键程度从大到小排序

    Keyword arguments:
    sentences         --  列表，元素是句子
    words             --  二维列表，子列表和sentences中的句子对应，子列表由单词组成
    sim_func          --  计算两个句子的相似性，参数是两个由单词组成的列表
    pagerank_config   --  pagerank的设置
    """
    sorted_sentences = []
    _source = words
    sentences_num = len(_source)        
    graph = np.zeros((sentences_num, sentences_num))
    
    for x in xrange(sentences_num):
        for y in xrange(x, sentences_num):
            similarity = sim_func( _source[x], _source[y] ) # 重点1
            graph[x, y] = similarity
            graph[y, x] = similarity
            
    nx_graph = nx.from_numpy_matrix(graph)
    scores = nx.pagerank(nx_graph, **pagerank_config)  # 重点2
    sorted_scores = sorted(scores.items(), key = lambda item: item[1], reverse=True)

    for index, score in sorted_scores:
        item = AttrDict(index=index, sentence=sentences[index], weight=score)
        sorted_sentences.append(item)

    return sorted_sentences
```

这边有两个重点，重点1：句子与句子的相似度是如何计算的；重点2：pagerank的实现。

很明显，PageRank 的实现是借助了 `networkx` 这个第三方库，在下一节我们会来看看这个第三方库的源码。

这边，我们先来看重点1，句子与句子的相似度是如何计算的，容易看出，计算方式和论文给的公式是一致的。

```python
# 文件：textrank4zh/util.py
# 行数：102
def get_similarity(word_list1, word_list2):
    """默认的用于计算两个句子相似度的函数。

    Keyword arguments:
    word_list1, word_list2  --  分别代表两个句子，都是由单词组成的列表
    """
    words   = list(set(word_list1 + word_list2))        
    vector1 = [float(word_list1.count(word)) for word in words]
    vector2 = [float(word_list2.count(word)) for word in words]
    
    vector3 = [vector1[x]*vector2[x]  for x in xrange(len(vector1))]
    vector4 = [1 for num in vector3 if num > 0.]
    co_occur_num = sum(vector4)

    if abs(co_occur_num) <= 1e-12:
        return 0.
    
    denominator = math.log(float(len(word_list1))) + math.log(float(len(word_list2))) # 分母
    
    if abs(denominator) < 1e-12:
        return 0.
    
    return co_occur_num / denominator
```

## 4. networkx 是怎么实现 PageRank的

不得不说，写 Python 的好处就是有各种第三方库可以用。整个PageRank的计算过程，大佬都借助了 `networkx` 这个第三方库。

`networkx` 中 PageRank 的路径为 `networkx/algorithms/link_analysis/pagerank_alg.py`。我这边就不贴出源码了，共476行，把我惊出一身冷汗。定睛一看，原来注释占了一半的行数。再仔细阅读一遍，原来写这个库的大佬用3种不同的方法实现了3个 PageRank 函数，请收下我的膝盖。

Python 的变量类型不明确，比如代码中 `W` 这个变量，我知道是一张图，但我不知道是用邻接矩阵还是邻接表或者是自定义类来表示的，需要向上回溯几层代码才能知道。所以阅读这种大工程的 Python 代码是需要花一点时间的。

如果有耐心理解源码的话，可以发现，`networkx` 中 PageRank 和论文中的数学公式还是有些不一样的，主要的不一样的点在于对 `dangling_nodes` 的处理。



## 5. 总结

写 Python 的好处就是有各种第三方库可以用。

Python 的变量类型不明确，阅读大工程的 Python 代码是需要花一点时间的。
