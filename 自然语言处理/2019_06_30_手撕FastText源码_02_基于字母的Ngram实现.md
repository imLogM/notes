# 手撕 FastText 源码（02）基于字母的 Ngram 实现（FastText's subwords）

作者：LogM

本文原载于 [https://segmentfault.com/u/logm/articles](https://segmentfault.com/u/logm/articles) ，不允许转载~

## 1. 源码来源

FastText 源码：[https://github.com/facebookresearch/fastText](https://github.com/facebookresearch/fastText)

本文对应的源码版本：Commits on Jun 27 2019, `979d8a9ac99c731d653843890c2364ade0f7d9d3`

FastText 论文：

[1] P. Bojanowski, E. Grave, A. Joulin, T. Mikolov, [*Enriching Word Vectors with Subword Information*](https://arxiv.org/abs/1607.04606)

[2] A. Joulin, E. Grave, P. Bojanowski, T. Mikolov, [*Bag of Tricks for Efficient Text Classification*](https://arxiv.org/abs/1607.01759)

## 2. 概述

[之前的博客](https://segmentfault.com/a/1190000019623774)介绍了"分类器的预测"的源码，里面有一个重点没有详细展开，就是"基于字母的 Ngram 是怎么实现的"。这块论文里面关于"字母Ngram的生成"讲的比较清楚，但是对于"字母Ngram"如何加入到模型中，讲的不太清楚，所以就求助于源码，源码里面把这块叫做 `Subwords`。

看懂了源码其实会发现 `Subwords` 加入到模型很简单，就是把它和"词语"一样对待，一起求和取平均。

另外，我自己再看源码的过程中还有个收获，就是关于"中文词怎么算subwords"，之前我一直觉得 `Subwords` 对中文无效，看了源码才知道是有影响的。

最后是词向量中怎么把 `Subwords` 加到模型。这部分我估计大家也不怎么关心，所以我就相当于写给我自己看的，解答自己看论文的疑惑。以`skipgram`为例，输入的 vector 和所要预测的 vector 都是`单个词语`与`subwords`相加求和的结果。

## 3. 怎么计算 `Subwords`

之前的博客有提到，`Dictionary::getLine` 这个函数的作用是从输入文件中读取一行，并将所有的Id（包括词语的Id，SubWords的Id，WordNgram的Id）存入到数组 `words` 中。

首先我们要来看看 `Subwords` 的 Id 是怎么生成的，对应的函数是 `Dictionary::addSubwords`。

```cpp
// 文件：src/dictionary.cc
// 行数：378
int32_t Dictionary::getLine(
    std::istream& in,                       // `in`是输入的文件
    std::vector<int32_t>& words,            // `words`是所有的Id组成的数组（包括词语的id，SubWords的Id，WordNgram的Id）
    std::vector<int32_t>& labels) const {   // 因为FastText支持多标签，所以这里的`labels`也是数组
  std::vector<int32_t> word_hashes;
  std::string token;
  int32_t ntokens = 0;

  reset(in);
  words.clear();
  labels.clear();
  while (readWord(in, token)) {              // `token` 是读到的一个词语，如果读到一行的行尾，则返回`EOF`
    uint32_t h = hash(token);                // 找到这个词语位于哪个hash桶
    int32_t wid = getId(token, h);           // 在hash桶中找到这个词语的Id，如果负数就是没找到对应的Id
    entry_type type = wid < 0 ? getType(token) : getType(wid);  // 如果没找到对应Id，则有可能是label，`getType`里会处理

    ntokens++;
    if (type == entry_type::word) {
      addSubwords(words, token, wid);         // 这个函数是我们要讲的重点
      word_hashes.push_back(h);
    } else if (type == entry_type::label && wid >= 0) {
      labels.push_back(wid - nwords_);
    }
    if (token == EOS) {
      break;
    }
  }
  addWordNgrams(words, word_hashes, args_->wordNgrams);
  return ntokens;
}
```

来到 `Dictionary::addSubwords`，可以看到重点是 `Dictionary::getSubwords`。

```cpp
// 文件：src/dictionary.cc
// 行数：325
void Dictionary::addSubwords(
    std::vector<int32_t>& line,         // 我们要把 `Subwords` 的 Id 插入到数组 `line` 中
    const std::string& token,           // `token` 是当前单词的字符串
    int32_t wid) const {                // `wid` 是当前单词的 Id
  if (wid < 0) { // out of vocab
    if (token != EOS) {
      computeSubwords(BOW + token + EOW, line);
    }
  } else {
    if (args_->maxn <= 0) { // in vocab w/o subwords   
      line.push_back(wid);              // 如果用户关闭了 `Subwords` 功能，则不计算
    } else { // in vocab w/ subwords
      const std::vector<int32_t>& ngrams = getSubwords(wid);    // 这句是重点，获取了 `Subwords` 对应的 Id
      line.insert(line.end(), ngrams.cbegin(), ngrams.cend());
    }
  }
}
```

来到 `Dictionary::getSubwords`，看来每个单词的 `subwords` 是事先计算好的。

```cpp
// 文件：src/dictionary.cc
// 行数：85
const std::vector<int32_t>& Dictionary::getSubwords(int32_t i) const {
  assert(i >= 0);
  assert(i < nwords_);
  return words_[i].subwords;
}
```

我们找找是哪里初始化了 `Subwords`。在 `Dictionary::initNgrams` 中。

```cpp
// 文件：src/dictionary.cc
// 行数：197
void Dictionary::initNgrams() {
  for (size_t i = 0; i < size_; i++) {
    std::string word = BOW + words_[i].word + EOW;    // 为单词增加开头和结尾符号，比如`where`变为`<where>`
    words_[i].subwords.clear();
    words_[i].subwords.push_back(i);                  // 论文里说了，这个单词本身也算是`subwords`的一种
    if (words_[i].word != EOS) {
      computeSubwords(word, words_[i].subwords);      // 这个是重点
    }
  }
}
```

来到`Dictionary::computeSubwords`。这边涉及到`UTF-8`的[编码](http://www.voidcn.com/article/p-tvjyezbc-bss.html)。

看懂了`UTF-8`的编码，应该是比较容易能理解这段代码的。这段代码的计算方式，和论文里给出的计算方式是一致的。

同时，这段代码也解答了"中文词怎么算subwords"的问题。

```cpp
// 文件：src/dictionary.cc
// 行数：172
void Dictionary::computeSubwords(
    const std::string& word,
    std::vector<int32_t>& ngrams,
    std::vector<std::string>* substrings=nullptr) const {
    for (size_t i = 0; i < word.size(); i++) {
        std::string ngram;
        if ((word[i] & 0xC0) == 0x80) {    // 和UTF-8的编码形式有关，判断是不是多字节编码的中间字节
            continue;
        }
        for (size_t j = i, n = 1; j < word.size() && n <= args_->maxn; n++) {
            ngram.push_back(word[j++]);
            while (j < word.size() && (word[j] & 0xC0) == 0x80) {
                ngram.push_back(word[j++]);
            }
            if (n >= args_->minn && !(n == 1 && (i == 0 || j == word.size()))) {
                int32_t h = hash(ngram) % args_->bucket;
                pushHash(ngrams, h);
                if (substrings) {
                    substrings->push_back(ngram);
                }
            }
        }
    }
}
```

## 4. `Subwords`是怎么加入到模型的

[之前的博客](https://segmentfault.com/a/1190000019623774)有提到，`Model::computeHidden`中的参数`input`就是Id组成的数组（包括词语的Id，SubWords的Id，WordNgram的Id）。现在主要看 `Vector::addRow` 是怎么实现的。

```cpp
// 文件：src/model.cc
// 行数：43
void Model::computeHidden(const std::vector<int32_t>& input, State& state)
    const {
  Vector& hidden = state.hidden;
  hidden.zero();
  for (auto it = input.cbegin(); it != input.cend(); ++it) {
    hidden.addRow(*wi_, *it);           // 求和，`wi_`是输入矩阵，要把`input`的每一项加到`wi_`上
  }
  hidden.mul(1.0 / input.size());       // 然后取平均
}
```

来到`Vector::addRow`，我们找到了重点是 `addRowToVector` 函数。但是这边用到了多态，需要分析下继承关系才知道 `addRowToVector` 函数位于哪个文件，我这边略过继承关系的分析过程，直接来到 `QuantMatrix::addRowToVector`。

```cpp
// 文件：src/vector.cc
// 行数：62
void Vector::addRow(const Matrix& A, int64_t i) {
  assert(i >= 0);
  assert(i < A.size(0));
  assert(size() == A.size(1));
  A.addRowToVector(*this, i);       // 这个是重点
}
```

来到 `QuantMatrix::addRowToVector`，代码有点涉及矩阵底层运算了，说实话，我没看懂。但我为什么知道是求和呢？因为论文说了，对于`单个词语`，hidden 层的操作是把单词的向量加和求平均，从这些代码看，`subwords`和`单个词语`是一致的，所以我才知道`subwords`也是加和求平均。

```cpp
// 文件：src/quantmatrix.cc
// 行数：75
void QuantMatrix::addRowToVector(Vector& x, int32_t i) const {
  real norm = 1;
  if (qnorm_) {
    norm = npq_->get_centroids(0, norm_codes_[i])[0];
  }
  pq_->addcode(x, codes_.data(), i, norm);
}
```

```cpp
// 文件：src/productquantizer.cc
// 行数：197
void ProductQuantizer::addcode(
    Vector& x,
    const uint8_t* codes,
    int32_t t,
    real alpha) const {
  auto d = dsub_;
  const uint8_t* code = codes + nsubq_ * t;
  for (auto m = 0; m < nsubq_; m++) {
    const real* c = get_centroids(m, code[m]);
    if (m == nsubq_ - 1) {
      d = lastdsub_;
    }
    for (auto n = 0; n < d; n++) {
      x[m * dsub_ + n] += alpha * c[n];
    }
  }
}
```

## 5. 词向量中的 `Subwords`

然后是词向量中怎么把 `Subwords` 加到模型。

这部分我估计大家也不怎么关心，所以我就相当于写给我自己看的，解答自己看论文的疑惑。以`skipgram`为例，输入的 vector 和所要预测的 vector 都是`单个词语`与`subwords`相加求和的结果。

```cpp
// 文件：src/productquantizer.cc
// 行数：393
void FastText::skipgram(
    Model::State& state,
    real lr,
    const std::vector<int32_t>& line) {
    std::uniform_int_distribution<> uniform(1, args_->ws);
    for (int32_t w = 0; w < line.size(); w++) {
        int32_t boundary = uniform(state.rng);
        const std::vector<int32_t>& ngrams = dict_->getSubwords(line[w]);    // 重点1
        for (int32_t c = -boundary; c <= boundary; c++) {
            if (c != 0 && w + c >= 0 && w + c < line.size()) {
                model_->update(ngrams, line, w + c, lr, state);       // 重点2
            }
        }
    }
}
```

```cpp
// 文件：src/model.cc
// 行数：70
void Model::update(
    const std::vector<int32_t>& input,
    const std::vector<int32_t>& targets,
    int32_t targetIndex,
    real lr,
    State& state) {
  if (input.size() == 0) {
    return;
  }
  computeHidden(input, state);

  Vector& grad = state.grad;
  grad.zero();
  real lossValue = loss_->forward(targets, targetIndex, state, lr, true);       // 重点
  state.incrementNExamples(lossValue);

  if (normalizeGradient_) {
    grad.mul(1.0 / input.size());
  }
  for (auto it = input.cbegin(); it != input.cend(); ++it) {
    wi_->addVectorToRow(grad, *it, 1.0);
  }
}
```

softmax的计算多看几遍应该还是能看明白的。梯度的计算，建议结合论文的公式看，加入 `subwords` 后，梯度的求取不再是softmax多分类的那种求梯度方式了。

```cpp
// 文件：src/loss.cc
// 行数：322
real SoftmaxLoss::forward(
    const std::vector<int32_t>& targets,
    int32_t targetIndex,
    Model::State& state,
    real lr,
    bool backprop) {
  computeOutput(state);     // 这个是重点

  assert(targetIndex >= 0);
  assert(targetIndex < targets.size());
  int32_t target = targets[targetIndex];

  if (backprop) {           // 计算梯度的过程，结合论文公式看
    int32_t osz = wo_->size(0);
    for (int32_t i = 0; i < osz; i++) {
      real label = (i == target) ? 1.0 : 0.0;
      real alpha = lr * (label - state.output[i]);
      state.grad.addRow(*wo_, i, alpha);
      wo_->addVectorToRow(state.hidden, i, alpha);
    }
  }
  return -log(state.output[target]);
};
```

```cpp
// 文件：src/loss.cc
// 行数：305
void SoftmaxLoss::computeOutput(Model::State& state) const {
  Vector& output = state.output;
  output.mul(*wo_, state.hidden);   // 结合论文公式看，和一般的softmax不一样
  real max = output[0], z = 0.0;
  int32_t osz = output.size();
  for (int32_t i = 0; i < osz; i++) {
    max = std::max(output[i], max);
  }
  for (int32_t i = 0; i < osz; i++) {
    output[i] = exp(output[i] - max);       // 应该是为了防止数值计算溢出，计算结果和原始的softmax公式一致
    z += output[i];
  }
  for (int32_t i = 0; i < osz; i++) {
    output[i] /= z;
  }
}
```