# 语言模型中文识字率分析

- [语言模型中文识字率分析](#语言模型中文识字率分析)
  - [项目介绍](#项目介绍)
  - [命令行工具 `vocab-coverage` 使用指南](#命令行工具-vocab-coverage-使用指南)
    - [安装](#安装)
    - [使用](#使用)
      - [`charset` 子命令](#charset-子命令)
      - [`coverage` 子命令](#coverage-子命令)
      - [`embedding` 子命令](#embedding-子命令)
  - [分析结果](#分析结果)
    - [BERT 类模型](#bert-类模型)
    - [T5 模型及其衍生的模型对比](#t5-模型及其衍生的模型对比)
    - [bert-base-chinese 模型及其衍生的模型对比](#bert-base-chinese-模型及其衍生的模型对比)
    - [ERNIE 模型及其衍生的模型对比](#ernie-模型及其衍生的模型对比)
    - [paraphrase-multilingual-MiniLM-L12-v2 模型及其衍生的模型对比](#paraphrase-multilingual-minilm-l12-v2-模型及其衍生的模型对比)
    - [xlm-roberta-base 模型及其微调后的模型对比](#xlm-roberta-base-模型及其微调后的模型对比)
    - [LLaMA 及衍生模型](#llama-及衍生模型)
    - [英文大语言模型](#英文大语言模型)
    - [中文大语言模型](#中文大语言模型)
    - [OpenAI 提供的模型](#openai-提供的模型)
    - [其他模型](#其他模型)

## 项目介绍

本项目的目的是为了调查各个语言模型的中文识字率的情况，以此可以作为后续模型评估分析的参考。

为了分析模型的中文识字率，我们使用三个常用的字符集，总共`21267`个汉字。

- 中华人民共和国教育部于2013年颁布的[《通用规范汉字表》](https://zh.wikipedia.org/zh-cn/%E9%80%9A%E7%94%A8%E8%A7%84%E8%8C%83%E6%B1%89%E5%AD%97%E8%A1%A8)，在该字表中，共收录了 `8105` 个汉字，其中一级字表（常用字集）`3500`个，二级字表`3000`个，三级字表`1605`个。字表内容从[中文百科](https://www.zwbk2009.com/)中获取。
- 中華民國教育部頒布的[《常用國字標準字體表》](https://zh.wikipedia.org/zh-hant/%E5%B8%B8%E7%94%A8%E5%9C%8B%E5%AD%97%E6%A8%99%E6%BA%96%E5%AD%97%E9%AB%94%E8%A1%A8) 中的甲表和乙表。甲表收录常用字`4808`个，其中有`1749`个汉字不在《通用规范汉字表》中；乙表收录次常用字`6343`个，其中有`4503`个汉字不在《通用规范汉字表》中。统计汉字识字率时，将只针对增加的汉字进行统计，已经在《通用规范汉字表》中的汉字不再重复统计。
- [《Unicode中日韩统一表意文字》](https://zh.wikipedia.org/zh-cn/%E4%B8%AD%E6%97%A5%E9%9F%93%E7%B5%B1%E4%B8%80%E8%A1%A8%E6%84%8F%E6%96%87%E5%AD%97_(Unicode%E5%8D%80%E6%AE%B5))，为汉字在 Unicode 中的基本区段。在 Unicode 14.0 时，收录了 `20992` 个汉字，占据码位 `U+4E00`-`U+9FFF`。其中有`6910`个汉字，既不在《通用规范汉字表》中，也不在《常用國字標準字體表》中。统计汉字识字率时，将只针对增加的汉字进行统计，已经在《通用规范汉字表》和《常用國字標準字體表》中的汉字不在重复统计。汉字在 Unicode 中还有其它区段，总共将近9万汉字，但由于其它汉字不常使用，这里暂不纳入统计范围。

对于语言模型是否认知某个汉字的判断，我们通过对应语言模型所使用的 Tokenizer 是否可以对该汉字进行 `encode` 来判断。

- 模型不认识某汉字的判定为：
  - 模型对该汉字的编码结果为空；
  - 模型对该汉字的编码结果为 `unk_token_id`；
- 模型认识某汉字的判定为：
  - 模型对该汉字的编码结果长度为1；
- 如果编码结果长度大于1，这有可能是因为使用了 BBPE 的原因，一个不常出现的汉字被拆分成了多个 token。由于汉字被以UTF-8的形式编码，拆散该编码并不能体现汉字语义，因此，一个汉字被打散的编码越多，我们认为该模型对该汉字的认知程度可能越低。所以，对于编码结果长度大于1的情况，我们认为该模型对该汉字的认知程度为 `1 / len(encode_result)`，用以控制半透明程度。在识字率的计数中，将计数为 `0`。

> 在进行判断前，会先行去除前缀后缀的特殊token。

## 命令行工具 `vocab-coverage` 使用指南

`vocab-coverage` 是一个命令行工具，用于分析模型的汉字识字率。

### 安装

```bash
pip install vocab-coverage
```

由于图中有汉字，因此需要中文字体，这里我使用了 `Noto Sans CJK` 字体用于中文，以及 `Anonymous Pro`字体，建议安装该字体。

**Linux**

如 Ubuntu，可以通过以下命令安装：

```bash
sudo apt install fonts-noto-cjk fonts-anonymous-pro
```

**Mac**

如 MacOS，可以通过以下命令安装：

```bash
brew install font-noto-sans-cjk font-noto-serif-cjk font-anonymous-pro
```

**Windows**

如 Windows，可以通过以下命令安装：

```bash
choco install noto anonymouspro
```

### 使用

`vocab-coverage` 它有三个子命令：`charset`、 `coverage` 和 `embedding`。

#### `charset` 子命令

`charset` 子命令用于生成用以统计识字率的字表文件。

```bash
$ vocab-coverage charset --help
usage: vocab-coverage charset [-h] [--charset_file CHARSET_FILE]

options:
  -h, --help            show this help message and exit
  --charset_file CHARSET_FILE
                        用以统计识字率的字表文件（默认：charset.json）
```

#### `coverage` 子命令

`coverage` 子命令用于分析模型的汉字识字率。

```bash
$ vocab-coverage coverage --help
usage: vocab-coverage coverage [-h] [--model_name MODEL_NAME] [--charset_file CHARSET_FILE] [--output_dir OUTPUT_DIR] [--debug]

options:
  -h, --help            show this help message and exit
  --model_name MODEL_NAME
                        模型在 HuggingFace Hub 上的名称（默认为 shibing624/text2vec-base-chinese）
  --charset_file CHARSET_FILE
                        用以统计识字率的字表文件（默认为内置字符集文件）
  --output_dir OUTPUT_DIR
                        生成的图像文件的输出目录（默认为 images）
  --debug               是否打印调试信息
```

- `--model_name`：模型在 HuggingFace Hub 上的名称。默认为 `shibing624/text2vec-base-chinese`。
- `--charset_file`：用以统计识字率的字表文件。默认为 `charset.json`。
- `--output_dir`：生成的图像文件的输出目录。默认为 `images`。
- `--debug`：是否打印调试信息。

**示例**

```bash
$ vocab-coverage coverage --model_name=THUDM/chatglm-6b
检查模型 THUDM/chatglm-6b 的字表
字表《通用规范汉字表》一级汉字：3499/3500 (99.97%)
字表《通用规范汉字表》二级汉字：1724/3000 (57.47%)
字表《通用规范汉字表》三级汉字：48/1605 (2.99%)
字表《常用國字標準字體表》甲表(增)：185/1749 (10.58%)
字表《常用國字標準字體表》乙表(增)：14/4503 (0.31%)
字表《Unicode中日韩统一表意文字》(增)：115/6910 (1.66%)
```

除了上述输出外，还会在 `images` 目录下生成一个图像文件，`images/coverage/THUDM_chatglm-6b.coverage.png`，为可视化的分析结果。

#### `embedding` 子命令

`embedding` 子命令用于分析模型词向量在空间中的分布情况。

```bash
usage: main.py embedding [-h] [--model_name MODEL_NAME] [--charset_file CHARSET_FILE] [--output_dir OUTPUT_DIR] [--is_detail] [--debug]

options:
  -h, --help            show this help message and exit
  --model_name MODEL_NAME
                        模型在 HuggingFace Hub 上的名称（默认为 shibing624/text2vec-base-chinese）
  --charset_file CHARSET_FILE
                        用以统计识字率的字表文件（默认为内置字符集文件）
  --output_dir OUTPUT_DIR
                        生成的图像文件的输出目录（默认为 images）
  --is_detail           是否对汉字进行详细分类（默认为 False）
  --debug               是否打印调试信息（默认为 False）
```

- `--model_name`：模型在 HuggingFace Hub 上的名称。默认为 `shibing624/text2vec-base-chinese`。
- `--charset_file`：用以加载字符集的文件。默认为 `charset.json`。
- `--output_dir`：生成的图像文件的输出目录。默认为 `images`。
- `--is_detail`：是否对汉字进行详细分类。默认为 `False`。
- `--debug`：是否打印调试信息。默认为 `False`。

**示例**

```bash
$ vocab-coverage embedding --model_name=THUDM/chatglm-6b --debug
对模型 THUDM/chatglm-6b 的 embedding 进行可视化...
Loading checkpoint shards: 100%|█████████████████████████████████████████████████████████████| 8/8 [00:14<00:00,  1.79s/it]
reducing embeddings (130528, 4096) to 2D...
[t-SNE] Computing 91 nearest neighbors...
[t-SNE] Indexed 130528 samples in 0.223s...
...
[t-SNE] Computed conditional probabilities for sample 130528 / 130528
[t-SNE] Mean sigma: 0.251563
[t-SNE] Computed conditional probabilities in 1.189s
[t-SNE] Iteration 50: error = 111.8378067, gradient norm = 0.0000213 (50 iterations in 8.560s)
[t-SNE] Iteration 100: error = 111.7450027, gradient norm = 0.0002728 (50 iterations in 13.605s)
[t-SNE] Iteration 150: error = 111.7283707, gradient norm = 0.0000039 (50 iterations in 7.279s)
[t-SNE] Iteration 200: error = 111.7283707, gradient norm = 0.0000039 (50 iterations in 7.123s)
...
[t-SNE] Iteration 950: error = 3.7165585, gradient norm = 0.0030942 (50 iterations in 7.229s)
[t-SNE] Iteration 1000: error = 3.6922786, gradient norm = 0.0029739 (50 iterations in 7.451s)
[t-SNE] KL divergence after 1000 iterations: 3.692279
draw embeddings (130528, 2)...
...
draw embedding point: 130528
font size: 25, font: ('Noto Sans CJK JP', 'Regular')
...
save to images/embeddings/THUDM-chatglm-6b.embedding.jpg...
```

## 分析结果

如果图片无法显示，请访问项目 Github 页面：<https://github.com/twang2218/vocab-coverage>

下面是挑选出来的一些比较有特点的模型，完整的模型分析列表请查看 [模型分析列表](graphs.md)。

### BERT 类模型

| 名称| ![](images/empty.png) 中文覆盖率 | ![](images/empty.png) 输入词向量分布 | ![](images/empty.png) 输出词向量分布 |
| :---: | :---: | :---: | :---: |
| <b>bert-base-cased</b> | ![Vocab Coverage for <b>bert-base-cased</b>](images/coverage/bert-base-cased.coverage.png) | [![input embedding image for <b>bert-base-cased</b>](images/thumbnails/bert-base-cased.embeddings.input.thumbnail.jpg)](images/embeddings/bert-base-cased.embeddings.input.jpg) | [![output embedding image for <b>bert-base-cased</b>](images/thumbnails/bert-base-cased.embeddings.output.thumbnail.jpg)](images/embeddings/bert-base-cased.embeddings.output.jpg) |
| <b>roberta-large</b> | ![Vocab Coverage for <b>roberta-large</b>](images/coverage/roberta-large.coverage.png) | [![input embedding image for <b>roberta-large</b>](images/thumbnails/roberta-large.embeddings.input.thumbnail.jpg)](images/embeddings/roberta-large.embeddings.input.jpg) | [![output embedding image for <b>roberta-large</b>](images/thumbnails/roberta-large.embeddings.output.thumbnail.jpg)](images/embeddings/roberta-large.embeddings.output.jpg) |
| <b>bert-base-multilingual-cased</b> | ![Vocab Coverage for <b>bert-base-multilingual-cased</b>](images/coverage/bert-base-multilingual-cased.coverage.png) | [![input embedding image for <b>bert-base-multilingual-cased</b>](images/thumbnails/bert-base-multilingual-cased.embeddings.input.thumbnail.jpg)](images/embeddings/bert-base-multilingual-cased.embeddings.input.jpg) | [![output embedding image for <b>bert-base-multilingual-cased</b>](images/thumbnails/bert-base-multilingual-cased.embeddings.output.thumbnail.jpg)](images/embeddings/bert-base-multilingual-cased.embeddings.output.jpg) |


### T5 模型及其衍生的模型对比

| 名称| ![](images/empty.png) 中文覆盖率 | ![](images/empty.png) 输入词向量分布 | ![](images/empty.png) 输出词向量分布 |
| :---: | :---: | :---: | :---: |
| <b><p>google</p><p>/</p><p>flan-t5-base</p></b> | ![Vocab Coverage for <b><p>google</p><p>/</p><p>flan-t5-base</p></b>](images/coverage/google_flan-t5-base.coverage.png) | [![input embedding image for <b><p>google</p><p>/</p><p>flan-t5-base</p></b>](images/thumbnails/google_flan-t5-base.embeddings.input.thumbnail.jpg)](images/embeddings/google_flan-t5-base.embeddings.input.jpg) | [![output embedding image for <b><p>google</p><p>/</p><p>flan-t5-base</p></b>](images/thumbnails/google_flan-t5-base.embeddings.output.thumbnail.jpg)](images/embeddings/google_flan-t5-base.embeddings.output.jpg) |
| <b><p>shibing624</p><p>/</p><p>prompt-t5-base-chinese</p></b> | ![Vocab Coverage for <b><p>shibing624</p><p>/</p><p>prompt-t5-base-chinese</p></b>](images/coverage/shibing624_prompt-t5-base-chinese.coverage.png) | [![input embedding image for <b><p>shibing624</p><p>/</p><p>prompt-t5-base-chinese</p></b>](images/thumbnails/shibing624_prompt-t5-base-chinese.embeddings.input.thumbnail.jpg)](images/embeddings/shibing624_prompt-t5-base-chinese.embeddings.input.jpg) | [![output embedding image for <b><p>shibing624</p><p>/</p><p>prompt-t5-base-chinese</p></b>](images/thumbnails/shibing624_prompt-t5-base-chinese.embeddings.output.thumbnail.jpg)](images/embeddings/shibing624_prompt-t5-base-chinese.embeddings.output.jpg) |
| <b><p>shibing624</p><p>/</p><p>mengzi-t5-base-chinese-correction</p></b> | ![Vocab Coverage for <b><p>shibing624</p><p>/</p><p>mengzi-t5-base-chinese-correction</p></b>](images/coverage/shibing624_mengzi-t5-base-chinese-correction.coverage.png) | [![input embedding image for <b><p>shibing624</p><p>/</p><p>mengzi-t5-base-chinese-correction</p></b>](images/thumbnails/shibing624_mengzi-t5-base-chinese-correction.embeddings.input.thumbnail.jpg)](images/embeddings/shibing624_mengzi-t5-base-chinese-correction.embeddings.input.jpg) | [![output embedding image for <b><p>shibing624</p><p>/</p><p>mengzi-t5-base-chinese-correction</p></b>](images/thumbnails/shibing624_mengzi-t5-base-chinese-correction.embeddings.output.thumbnail.jpg)](images/embeddings/shibing624_mengzi-t5-base-chinese-correction.embeddings.output.jpg) |


### bert-base-chinese 模型及其衍生的模型对比

| 名称| ![](images/empty.png) 中文覆盖率 | ![](images/empty.png) 输入词向量分布 | ![](images/empty.png) 输出词向量分布 |
| :---: | :---: | :---: | :---: |
| <b>bert-base-chinese</b> | ![Vocab Coverage for <b>bert-base-chinese</b>](images/coverage/bert-base-chinese.coverage.png) | [![input embedding image for <b>bert-base-chinese</b>](images/thumbnails/bert-base-chinese.embeddings.input.thumbnail.jpg)](images/embeddings/bert-base-chinese.embeddings.input.jpg) | [![output embedding image for <b>bert-base-chinese</b>](images/thumbnails/bert-base-chinese.embeddings.output.thumbnail.jpg)](images/embeddings/bert-base-chinese.embeddings.output.jpg) |
| <b><p>hfl</p><p>/</p><p>chinese-macbert-base</p></b> | ![Vocab Coverage for <b><p>hfl</p><p>/</p><p>chinese-macbert-base</p></b>](images/coverage/hfl_chinese-macbert-base.coverage.png) | [![input embedding image for <b><p>hfl</p><p>/</p><p>chinese-macbert-base</p></b>](images/thumbnails/hfl_chinese-macbert-base.embeddings.input.thumbnail.jpg)](images/embeddings/hfl_chinese-macbert-base.embeddings.input.jpg) | [![output embedding image for <b><p>hfl</p><p>/</p><p>chinese-macbert-base</p></b>](images/thumbnails/hfl_chinese-macbert-base.embeddings.output.thumbnail.jpg)](images/embeddings/hfl_chinese-macbert-base.embeddings.output.jpg) |
| <b><p>shibing624</p><p>/</p><p>text2vec-base-chinese</p></b> | ![Vocab Coverage for <b><p>shibing624</p><p>/</p><p>text2vec-base-chinese</p></b>](images/coverage/shibing624_text2vec-base-chinese.coverage.png) | [![input embedding image for <b><p>shibing624</p><p>/</p><p>text2vec-base-chinese</p></b>](images/thumbnails/shibing624_text2vec-base-chinese.embeddings.input.thumbnail.jpg)](images/embeddings/shibing624_text2vec-base-chinese.embeddings.input.jpg) | [![output embedding image for <b><p>shibing624</p><p>/</p><p>text2vec-base-chinese</p></b>](images/thumbnails/shibing624_text2vec-base-chinese.embeddings.output.thumbnail.jpg)](images/embeddings/shibing624_text2vec-base-chinese.embeddings.output.jpg) |
| <b><p>moka-ai</p><p>/</p><p>m3e-base</p></b> | ![Vocab Coverage for <b><p>moka-ai</p><p>/</p><p>m3e-base</p></b>](images/coverage/moka-ai_m3e-base.coverage.png) | [![input embedding image for <b><p>moka-ai</p><p>/</p><p>m3e-base</p></b>](images/thumbnails/moka-ai_m3e-base.embeddings.input.thumbnail.jpg)](images/embeddings/moka-ai_m3e-base.embeddings.input.jpg) | [![output embedding image for <b><p>moka-ai</p><p>/</p><p>m3e-base</p></b>](images/thumbnails/moka-ai_m3e-base.embeddings.output.thumbnail.jpg)](images/embeddings/moka-ai_m3e-base.embeddings.output.jpg) |
| <b><p>junnyu</p><p>/</p><p>wobert_chinese_plus_base</p></b> | ![Vocab Coverage for <b><p>junnyu</p><p>/</p><p>wobert_chinese_plus_base</p></b>](images/coverage/junnyu_wobert_chinese_plus_base.coverage.png) | [![input embedding image for <b><p>junnyu</p><p>/</p><p>wobert_chinese_plus_base</p></b>](images/thumbnails/junnyu_wobert_chinese_plus_base.embeddings.input.thumbnail.jpg)](images/embeddings/junnyu_wobert_chinese_plus_base.embeddings.input.jpg) | [![output embedding image for <b><p>junnyu</p><p>/</p><p>wobert_chinese_plus_base</p></b>](images/thumbnails/junnyu_wobert_chinese_plus_base.embeddings.output.thumbnail.jpg)](images/embeddings/junnyu_wobert_chinese_plus_base.embeddings.output.jpg) |


### ERNIE 模型及其衍生的模型对比

| 名称| ![](images/empty.png) 中文覆盖率 | ![](images/empty.png) 输入词向量分布 | ![](images/empty.png) 输出词向量分布 |
| :---: | :---: | :---: | :---: |
| <b><p>nghuyong</p><p>/</p><p>ernie-3.0-base-zh</p></b> | ![Vocab Coverage for <b><p>nghuyong</p><p>/</p><p>ernie-3.0-base-zh</p></b>](images/coverage/nghuyong_ernie-3.0-base-zh.coverage.png) | [![input embedding image for <b><p>nghuyong</p><p>/</p><p>ernie-3.0-base-zh</p></b>](images/thumbnails/nghuyong_ernie-3.0-base-zh.embeddings.input.thumbnail.jpg)](images/embeddings/nghuyong_ernie-3.0-base-zh.embeddings.input.jpg) | [![output embedding image for <b><p>nghuyong</p><p>/</p><p>ernie-3.0-base-zh</p></b>](images/thumbnails/nghuyong_ernie-3.0-base-zh.embeddings.output.thumbnail.jpg)](images/embeddings/nghuyong_ernie-3.0-base-zh.embeddings.output.jpg) |
| <b><p>shibing624</p><p>/</p><p>text2vec-base-chinese-sentence</p></b> | ![Vocab Coverage for <b><p>shibing624</p><p>/</p><p>text2vec-base-chinese-sentence</p></b>](images/coverage/shibing624_text2vec-base-chinese-sentence.coverage.png) | [![input embedding image for <b><p>shibing624</p><p>/</p><p>text2vec-base-chinese-sentence</p></b>](images/thumbnails/shibing624_text2vec-base-chinese-sentence.embeddings.input.thumbnail.jpg)](images/embeddings/shibing624_text2vec-base-chinese-sentence.embeddings.input.jpg) | [![output embedding image for <b><p>shibing624</p><p>/</p><p>text2vec-base-chinese-sentence</p></b>](images/thumbnails/shibing624_text2vec-base-chinese-sentence.embeddings.output.thumbnail.jpg)](images/embeddings/shibing624_text2vec-base-chinese-sentence.embeddings.output.jpg) |
| <b><p>shibing624</p><p>/</p><p>text2vec-base-chinese-paraphrase</p></b> | ![Vocab Coverage for <b><p>shibing624</p><p>/</p><p>text2vec-base-chinese-paraphrase</p></b>](images/coverage/shibing624_text2vec-base-chinese-paraphrase.coverage.png) | [![input embedding image for <b><p>shibing624</p><p>/</p><p>text2vec-base-chinese-paraphrase</p></b>](images/thumbnails/shibing624_text2vec-base-chinese-paraphrase.embeddings.input.thumbnail.jpg)](images/embeddings/shibing624_text2vec-base-chinese-paraphrase.embeddings.input.jpg) | [![output embedding image for <b><p>shibing624</p><p>/</p><p>text2vec-base-chinese-paraphrase</p></b>](images/thumbnails/shibing624_text2vec-base-chinese-paraphrase.embeddings.output.thumbnail.jpg)](images/embeddings/shibing624_text2vec-base-chinese-paraphrase.embeddings.output.jpg) |


### paraphrase-multilingual-MiniLM-L12-v2 模型及其衍生的模型对比

| 名称| ![](images/empty.png) 中文覆盖率 | ![](images/empty.png) 输入词向量分布 | ![](images/empty.png) 输出词向量分布 |
| :---: | :---: | :---: | :---: |
| <b><p>sentence-transformers</p><p>/</p><p>paraphrase-multilingual-MiniLM-L12-v2</p></b> | ![Vocab Coverage for <b><p>sentence-transformers</p><p>/</p><p>paraphrase-multilingual-MiniLM-L12-v2</p></b>](images/coverage/sentence-transformers_paraphrase-multilingual-MiniLM-L12-v2.coverage.png) | [![input embedding image for <b><p>sentence-transformers</p><p>/</p><p>paraphrase-multilingual-MiniLM-L12-v2</p></b>](images/thumbnails/sentence-transformers_paraphrase-multilingual-MiniLM-L12-v2.embeddings.input.thumbnail.jpg)](images/embeddings/sentence-transformers_paraphrase-multilingual-MiniLM-L12-v2.embeddings.input.jpg) | [![output embedding image for <b><p>sentence-transformers</p><p>/</p><p>paraphrase-multilingual-MiniLM-L12-v2</p></b>](images/thumbnails/sentence-transformers_paraphrase-multilingual-MiniLM-L12-v2.embeddings.output.thumbnail.jpg)](images/embeddings/sentence-transformers_paraphrase-multilingual-MiniLM-L12-v2.embeddings.output.jpg) |
| <b><p>shibing624</p><p>/</p><p>text2vec-base-multilingual</p></b> | ![Vocab Coverage for <b><p>shibing624</p><p>/</p><p>text2vec-base-multilingual</p></b>](images/coverage/shibing624_text2vec-base-multilingual.coverage.png) | [![input embedding image for <b><p>shibing624</p><p>/</p><p>text2vec-base-multilingual</p></b>](images/thumbnails/shibing624_text2vec-base-multilingual.embeddings.input.thumbnail.jpg)](images/embeddings/shibing624_text2vec-base-multilingual.embeddings.input.jpg) | [![output embedding image for <b><p>shibing624</p><p>/</p><p>text2vec-base-multilingual</p></b>](images/thumbnails/shibing624_text2vec-base-multilingual.embeddings.output.thumbnail.jpg)](images/embeddings/shibing624_text2vec-base-multilingual.embeddings.output.jpg) |


### xlm-roberta-base 模型及其微调后的模型对比

| 名称| ![](images/empty.png) 中文覆盖率 | ![](images/empty.png) 输入词向量分布 | ![](images/empty.png) 输出词向量分布 |
| :---: | :---: | :---: | :---: |
| <b>xlm-roberta-base</b> | ![Vocab Coverage for <b>xlm-roberta-base</b>](images/coverage/xlm-roberta-base.coverage.png) | [![input embedding image for <b>xlm-roberta-base</b>](images/thumbnails/xlm-roberta-base.embeddings.input.thumbnail.jpg)](images/embeddings/xlm-roberta-base.embeddings.input.jpg) | [![output embedding image for <b>xlm-roberta-base</b>](images/thumbnails/xlm-roberta-base.embeddings.output.thumbnail.jpg)](images/embeddings/xlm-roberta-base.embeddings.output.jpg) |
| <b><p>sentence-transformers</p><p>/</p><p>paraphrase-multilingual-mpnet-base-v2</p></b> | ![Vocab Coverage for <b><p>sentence-transformers</p><p>/</p><p>paraphrase-multilingual-mpnet-base-v2</p></b>](images/coverage/sentence-transformers_paraphrase-multilingual-mpnet-base-v2.coverage.png) | [![input embedding image for <b><p>sentence-transformers</p><p>/</p><p>paraphrase-multilingual-mpnet-base-v2</p></b>](images/thumbnails/sentence-transformers_paraphrase-multilingual-mpnet-base-v2.embeddings.input.thumbnail.jpg)](images/embeddings/sentence-transformers_paraphrase-multilingual-mpnet-base-v2.embeddings.input.jpg) | [![output embedding image for <b><p>sentence-transformers</p><p>/</p><p>paraphrase-multilingual-mpnet-base-v2</p></b>](images/thumbnails/sentence-transformers_paraphrase-multilingual-mpnet-base-v2.embeddings.output.thumbnail.jpg)](images/embeddings/sentence-transformers_paraphrase-multilingual-mpnet-base-v2.embeddings.output.jpg) |


### LLaMA 及衍生模型

| 名称| ![](images/empty.png) 中文覆盖率 | ![](images/empty.png) 输入词向量分布 | ![](images/empty.png) 输出词向量分布 |
| :---: | :---: | :---: | :---: |
| <b><p>decapoda-research</p><p>/</p><p>llama-7b-hf</p></b> | ![Vocab Coverage for <b><p>decapoda-research</p><p>/</p><p>llama-7b-hf</p></b>](images/coverage/decapoda-research_llama-7b-hf.coverage.png) | [![input embedding image for <b><p>decapoda-research</p><p>/</p><p>llama-7b-hf</p></b>](images/thumbnails/decapoda-research_llama-7b-hf.embeddings.input.thumbnail.jpg)](images/embeddings/decapoda-research_llama-7b-hf.embeddings.input.jpg) | [![output embedding image for <b><p>decapoda-research</p><p>/</p><p>llama-7b-hf</p></b>](images/thumbnails/decapoda-research_llama-7b-hf.embeddings.output.thumbnail.jpg)](images/embeddings/decapoda-research_llama-7b-hf.embeddings.output.jpg) |
| <b><p>lmsys</p><p>/</p><p>vicuna-7b-delta-v1.1</p></b> | ![Vocab Coverage for <b><p>lmsys</p><p>/</p><p>vicuna-7b-delta-v1.1</p></b>](images/coverage/lmsys_vicuna-7b-delta-v1.1.coverage.png) | [![input embedding image for <b><p>lmsys</p><p>/</p><p>vicuna-7b-delta-v1.1</p></b>](images/thumbnails/lmsys_vicuna-7b-delta-v1.1.embeddings.input.thumbnail.jpg)](images/embeddings/lmsys_vicuna-7b-delta-v1.1.embeddings.input.jpg) | [![output embedding image for <b><p>lmsys</p><p>/</p><p>vicuna-7b-delta-v1.1</p></b>](images/thumbnails/lmsys_vicuna-7b-delta-v1.1.embeddings.output.thumbnail.jpg)](images/embeddings/lmsys_vicuna-7b-delta-v1.1.embeddings.output.jpg) |
| <b><p>togethercomputer</p><p>/</p><p>RedPajama-INCITE-7B-Chat</p></b> | ![Vocab Coverage for <b><p>togethercomputer</p><p>/</p><p>RedPajama-INCITE-7B-Chat</p></b>](images/coverage/togethercomputer_RedPajama-INCITE-7B-Chat.coverage.png) | [![input embedding image for <b><p>togethercomputer</p><p>/</p><p>RedPajama-INCITE-7B-Chat</p></b>](images/thumbnails/togethercomputer_RedPajama-INCITE-7B-Chat.embeddings.input.thumbnail.jpg)](images/embeddings/togethercomputer_RedPajama-INCITE-7B-Chat.embeddings.input.jpg) | [![output embedding image for <b><p>togethercomputer</p><p>/</p><p>RedPajama-INCITE-7B-Chat</p></b>](images/thumbnails/togethercomputer_RedPajama-INCITE-7B-Chat.embeddings.output.thumbnail.jpg)](images/embeddings/togethercomputer_RedPajama-INCITE-7B-Chat.embeddings.output.jpg) |
| <b><p>openlm-research</p><p>/</p><p>open_llama_7b</p></b> | ![Vocab Coverage for <b><p>openlm-research</p><p>/</p><p>open_llama_7b</p></b>](images/coverage/openlm-research_open_llama_7b.coverage.png) | [![input embedding image for <b><p>openlm-research</p><p>/</p><p>open_llama_7b</p></b>](images/thumbnails/openlm-research_open_llama_7b.embeddings.input.thumbnail.jpg)](images/embeddings/openlm-research_open_llama_7b.embeddings.input.jpg) | [![output embedding image for <b><p>openlm-research</p><p>/</p><p>open_llama_7b</p></b>](images/thumbnails/openlm-research_open_llama_7b.embeddings.output.thumbnail.jpg)](images/embeddings/openlm-research_open_llama_7b.embeddings.output.jpg) |
| <b><p>shibing624</p><p>/</p><p>chinese-alpaca-plus-7b-hf</p></b> | ![Vocab Coverage for <b><p>shibing624</p><p>/</p><p>chinese-alpaca-plus-7b-hf</p></b>](images/coverage/shibing624_chinese-alpaca-plus-7b-hf.coverage.png) | [![input embedding image for <b><p>shibing624</p><p>/</p><p>chinese-alpaca-plus-7b-hf</p></b>](images/thumbnails/shibing624_chinese-alpaca-plus-7b-hf.embeddings.input.thumbnail.jpg)](images/embeddings/shibing624_chinese-alpaca-plus-7b-hf.embeddings.input.jpg) | [![output embedding image for <b><p>shibing624</p><p>/</p><p>chinese-alpaca-plus-7b-hf</p></b>](images/thumbnails/shibing624_chinese-alpaca-plus-7b-hf.embeddings.output.thumbnail.jpg)](images/embeddings/shibing624_chinese-alpaca-plus-7b-hf.embeddings.output.jpg) |


### 英文大语言模型

| 名称| ![](images/empty.png) 中文覆盖率 | ![](images/empty.png) 输入词向量分布 | ![](images/empty.png) 输出词向量分布 |
| :---: | :---: | :---: | :---: |
| <b><p>decapoda-research</p><p>/</p><p>llama-7b-hf</p></b> | ![Vocab Coverage for <b><p>decapoda-research</p><p>/</p><p>llama-7b-hf</p></b>](images/coverage/decapoda-research_llama-7b-hf.coverage.png) | [![input embedding image for <b><p>decapoda-research</p><p>/</p><p>llama-7b-hf</p></b>](images/thumbnails/decapoda-research_llama-7b-hf.embeddings.input.thumbnail.jpg)](images/embeddings/decapoda-research_llama-7b-hf.embeddings.input.jpg) | [![output embedding image for <b><p>decapoda-research</p><p>/</p><p>llama-7b-hf</p></b>](images/thumbnails/decapoda-research_llama-7b-hf.embeddings.output.thumbnail.jpg)](images/embeddings/decapoda-research_llama-7b-hf.embeddings.output.jpg) |
| <b><p>mosaicml</p><p>/</p><p>mpt-7b-instruct</p></b> | ![Vocab Coverage for <b><p>mosaicml</p><p>/</p><p>mpt-7b-instruct</p></b>](images/coverage/mosaicml_mpt-7b-instruct.coverage.png) | [![input embedding image for <b><p>mosaicml</p><p>/</p><p>mpt-7b-instruct</p></b>](images/thumbnails/mosaicml_mpt-7b-instruct.embeddings.input.thumbnail.jpg)](images/embeddings/mosaicml_mpt-7b-instruct.embeddings.input.jpg) | [![output embedding image for <b><p>mosaicml</p><p>/</p><p>mpt-7b-instruct</p></b>](images/thumbnails/mosaicml_mpt-7b-instruct.embeddings.output.thumbnail.jpg)](images/embeddings/mosaicml_mpt-7b-instruct.embeddings.output.jpg) |
| <b><p>tiiuae</p><p>/</p><p>falcon-7b-instruct</p></b> | ![Vocab Coverage for <b><p>tiiuae</p><p>/</p><p>falcon-7b-instruct</p></b>](images/coverage/tiiuae_falcon-7b-instruct.coverage.png) | [![input embedding image for <b><p>tiiuae</p><p>/</p><p>falcon-7b-instruct</p></b>](images/thumbnails/tiiuae_falcon-7b-instruct.embeddings.input.thumbnail.jpg)](images/embeddings/tiiuae_falcon-7b-instruct.embeddings.input.jpg) | [![output embedding image for <b><p>tiiuae</p><p>/</p><p>falcon-7b-instruct</p></b>](images/thumbnails/tiiuae_falcon-7b-instruct.embeddings.output.thumbnail.jpg)](images/embeddings/tiiuae_falcon-7b-instruct.embeddings.output.jpg) |
| <b><p>nomic-ai</p><p>/</p><p>gpt4all-j</p></b> | ![Vocab Coverage for <b><p>nomic-ai</p><p>/</p><p>gpt4all-j</p></b>](images/coverage/nomic-ai_gpt4all-j.coverage.png) | [![input embedding image for <b><p>nomic-ai</p><p>/</p><p>gpt4all-j</p></b>](images/thumbnails/nomic-ai_gpt4all-j.embeddings.input.thumbnail.jpg)](images/embeddings/nomic-ai_gpt4all-j.embeddings.input.jpg) | [![output embedding image for <b><p>nomic-ai</p><p>/</p><p>gpt4all-j</p></b>](images/thumbnails/nomic-ai_gpt4all-j.embeddings.output.thumbnail.jpg)](images/embeddings/nomic-ai_gpt4all-j.embeddings.output.jpg) |
| <b><p>OpenAssistant</p><p>/</p><p>oasst-sft-4-pythia-12b-epoch-3.5</p></b> | ![Vocab Coverage for <b><p>OpenAssistant</p><p>/</p><p>oasst-sft-4-pythia-12b-epoch-3.5</p></b>](images/coverage/OpenAssistant_oasst-sft-4-pythia-12b-epoch-3.5.coverage.png) | [![input embedding image for <b><p>OpenAssistant</p><p>/</p><p>oasst-sft-4-pythia-12b-epoch-3.5</p></b>](images/thumbnails/OpenAssistant_oasst-sft-4-pythia-12b-epoch-3.5.embeddings.input.thumbnail.jpg)](images/embeddings/OpenAssistant_oasst-sft-4-pythia-12b-epoch-3.5.embeddings.input.jpg) | [![output embedding image for <b><p>OpenAssistant</p><p>/</p><p>oasst-sft-4-pythia-12b-epoch-3.5</p></b>](images/thumbnails/OpenAssistant_oasst-sft-4-pythia-12b-epoch-3.5.embeddings.output.thumbnail.jpg)](images/embeddings/OpenAssistant_oasst-sft-4-pythia-12b-epoch-3.5.embeddings.output.jpg) |


### 中文大语言模型

| 名称| ![](images/empty.png) 中文覆盖率 | ![](images/empty.png) 输入词向量分布 | ![](images/empty.png) 输出词向量分布 |
| :---: | :---: | :---: | :---: |
| <b><p>THUDM</p><p>/</p><p>chatglm-6b</p></b> | ![Vocab Coverage for <b><p>THUDM</p><p>/</p><p>chatglm-6b</p></b>](images/coverage/THUDM_chatglm-6b.coverage.png) | [![input embedding image for <b><p>THUDM</p><p>/</p><p>chatglm-6b</p></b>](images/thumbnails/THUDM_chatglm-6b.embeddings.input.thumbnail.jpg)](images/embeddings/THUDM_chatglm-6b.embeddings.input.jpg) | [![output embedding image for <b><p>THUDM</p><p>/</p><p>chatglm-6b</p></b>](images/thumbnails/THUDM_chatglm-6b.embeddings.output.thumbnail.jpg)](images/embeddings/THUDM_chatglm-6b.embeddings.output.jpg) |
| <b><p>THUDM</p><p>/</p><p>chatglm2-6b</p></b> | ![Vocab Coverage for <b><p>THUDM</p><p>/</p><p>chatglm2-6b</p></b>](images/coverage/THUDM_chatglm2-6b.coverage.png) | [![input embedding image for <b><p>THUDM</p><p>/</p><p>chatglm2-6b</p></b>](images/thumbnails/THUDM_chatglm2-6b.embeddings.input.thumbnail.jpg)](images/embeddings/THUDM_chatglm2-6b.embeddings.input.jpg) | [![output embedding image for <b><p>THUDM</p><p>/</p><p>chatglm2-6b</p></b>](images/thumbnails/THUDM_chatglm2-6b.embeddings.output.thumbnail.jpg)](images/embeddings/THUDM_chatglm2-6b.embeddings.output.jpg) |
| <b><p>fnlp</p><p>/</p><p>moss-moon-003-sft</p></b> | ![Vocab Coverage for <b><p>fnlp</p><p>/</p><p>moss-moon-003-sft</p></b>](images/coverage/fnlp_moss-moon-003-sft.coverage.png) | [![input embedding image for <b><p>fnlp</p><p>/</p><p>moss-moon-003-sft</p></b>](images/thumbnails/fnlp_moss-moon-003-sft.embeddings.input.thumbnail.jpg)](images/embeddings/fnlp_moss-moon-003-sft.embeddings.input.jpg) | [![output embedding image for <b><p>fnlp</p><p>/</p><p>moss-moon-003-sft</p></b>](images/thumbnails/fnlp_moss-moon-003-sft.embeddings.output.thumbnail.jpg)](images/embeddings/fnlp_moss-moon-003-sft.embeddings.output.jpg) |
| <b><p>baichuan-inc</p><p>/</p><p>baichuan-7B</p></b> | ![Vocab Coverage for <b><p>baichuan-inc</p><p>/</p><p>baichuan-7B</p></b>](images/coverage/baichuan-inc_baichuan-7B.coverage.png) | [![input embedding image for <b><p>baichuan-inc</p><p>/</p><p>baichuan-7B</p></b>](images/thumbnails/baichuan-inc_baichuan-7B.embeddings.input.thumbnail.jpg)](images/embeddings/baichuan-inc_baichuan-7B.embeddings.input.jpg) | [![output embedding image for <b><p>baichuan-inc</p><p>/</p><p>baichuan-7B</p></b>](images/thumbnails/baichuan-inc_baichuan-7B.embeddings.output.thumbnail.jpg)](images/embeddings/baichuan-inc_baichuan-7B.embeddings.output.jpg) |


### OpenAI 提供的模型

| 名称| ![](images/empty.png) 中文覆盖率 | ![](images/empty.png) 输入词向量分布 | ![](images/empty.png) 输出词向量分布 |
| :---: | :---: | :---: | :---: |
| <b><p>OpenAI</p><p>/</p><p>text-embedding-ada-002</p></b> | ![Vocab Coverage for <b><p>OpenAI</p><p>/</p><p>text-embedding-ada-002</p></b>](images/coverage/OpenAI_text-embedding-ada-002.coverage.png) |   | [![output embedding image for <b><p>OpenAI</p><p>/</p><p>text-embedding-ada-002</p></b>](images/thumbnails/OpenAI_text-embedding-ada-002.embeddings.output.thumbnail.jpg)](images/embeddings/OpenAI_text-embedding-ada-002.embeddings.output.jpg) |




### 其他模型

请参见 [模型分析列表](graphs.md)。
