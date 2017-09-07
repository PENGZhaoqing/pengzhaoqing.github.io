---
date: 2017-07-10
title:  RNN聊天机器人与Beam Search [Tensorflow Seq2Seq]
categories: 深度学习
tags: [ruby]
---


本博客分析了一个Tensorflow实现的开源聊天机器人项目deepQA，首先从数据集上和一些重要代码上进行了说明和阐述，最后针对于测试的情况，在deepQA项目上实现了Beam Search的方法，让模型输出的句子更加准确。

## DeepQA
DeepQA是一个Tensorflow实现的开源的seq2seq模型的聊天机器人( [传送们](https://github.com/Conchylicultor/DeepQA)),出自谷歌的一篇关于对话模型的论文[A Neural Conversational Model](https://arxiv.org/abs/1506.05869)，训练的语料库包含电影台词的对话（Cornell和扩展版本的Cornell），Scotus对话库，以及Ubantu的对话。这些数据都能在项目的data里找到，但是目前好像只能针对某一个对话数据库进行训练，还没有支持混合对话的训练。目前实现的模型使用的是基础RNN中的seq2seq模型，主要针对的是比较短的对话。

### 1. Cornell数据集

DeepQA默认的是Cornell对话数据，一共两个文件：人物对话信息[movie_conversations.txt](https://github.com/PENGZhaoqing/DeepQA/blob/master/data/cornell/movie_conversations.txt)和具体对话内容[movie_lines.txt](https://github.com/PENGZhaoqing/DeepQA/blob/master/data/cornell/movie_lines.txt)，`+++$+++`为分隔符，movie_conversations.txt里每一行的第一个数据代表对话人物1的ID，第二个数据代表对话人物2的ID，第三个数据代表电影ID，后面是对话的ID，而movie_lines.txt里每一行的第一个数据代表对话ID，第二个数据表示说话的人物ID，第三个数据电影ID，第四个是此人物的名字，最后是这句话的具体内容。
```
＃ movie_conversations.txt
u0 +++$+++ u2 +++$+++ m0 +++$+++ ['L194', 'L195', 'L196', 'L197']
u0 +++$+++ u2 +++$+++ m0 +++$+++ ['L198', 'L199']
u0 +++$+++ u2 +++$+++ m0 +++$+++ ['L200', 'L201', 'L202', 'L203']
u0 +++$+++ u2 +++$+++ m0 +++$+++ ['L204', 'L205', 'L206']
u0 +++$+++ u2 +++$+++ m0 +++$+++ ['L207', 'L208']
u0 +++$+++ u2 +++$+++ m0 +++$+++ ['L271', 'L272', 'L273', 'L274', 'L275']
u0 +++$+++ u2 +++$+++ m0 +++$+++ ['L276', 'L277']

＃movie_lines.txt
L1045 +++$+++ u0 +++$+++ m0 +++$+++ BIANCA +++$+++ They do not!
L1044 +++$+++ u2 +++$+++ m0 +++$+++ CAMERON +++$+++ They do to!
L985 +++$+++ u0 +++$+++ m0 +++$+++ BIANCA +++$+++ I hope so.
L984 +++$+++ u2 +++$+++ m0 +++$+++ CAMERON +++$+++ She okay?
L925 +++$+++ u0 +++$+++ m0 +++$+++ BIANCA +++$+++ Let's go.
L924 +++$+++ u2 +++$+++ m0 +++$+++ CAMERON +++$+++ Wow
L872 +++$+++ u0 +++$+++ m0 +++$+++ BIANCA +++$+++ Okay -- you're gonna need to learn how to lie.
L871 +++$+++ u2 +++$+++ m0 +++$+++ CAMERON +++$+++ No
L870 +++$+++ u0 +++$+++ m0 +++$+++ BIANCA +++$+++ I'm kidding.  You know how sometimes you just become this "persona"?  And you don't know how to quit?
L869 +++$+++ u0 +++$+++ m0 +++$+++ BIANCA +++$+++ Like my fear of wearing pastels?
L868 +++$+++ u2 +++$+++ m0 +++$+++ CAMERON +++$+++ The "real you".
```

通过解析这个两个文件，可以在data/samples路径生成dataset-cornell.pkl数据集以及dataset-cornel-length10-filter1-vocabSize40000.pkl词汇表，默认对话的长度一般最大为10，过滤词频小于1的词并用unk代替，词汇表最大40000；这些参数可以自行修改，生成过程具体由textdata.py实现

### 2. 重要代码分析


####  2.1 构建基础RNN网络结构

直接看一下构建模型的代码，首先定义单个LSTM cell，然后用Dropout包裹，最后用参数numLayers决定多少层Stack结构的RNN：
```
def create_rnn_cell():
    encoDecoCell = tf.contrib.rnn.BasicLSTMCell(  # Or GRUCell, LSTMCell(args.hiddenSize)
        self.args.hiddenSize,
    )
    if not self.args.test:  # TODO: Should use a placeholder instead
        encoDecoCell = tf.contrib.rnn.DropoutWrapper(
            encoDecoCell,
            input_keep_prob=1.0,
            output_keep_prob=self.args.dropout
        )
    return encoDecoCell

encoDecoCell = tf.contrib.rnn.MultiRNNCell(
    [create_rnn_cell() for _ in range(self.args.numLayers)],
)
```

#### 2.2 定义输入值

接着定义网络的输入值，根据标准的seq2seq模型，一共四个：
1. encorder的输入：人物1说的一句话A，最大长度10
2. decoder的输入：人物2回复的对话B，因为前后分别加上了go开始符和end结束符，最大长度为12
3. decoder的target输入：decoder输入的目标输出，与decoder的输入一样但只有end标示符号，可以理解为decoder的输入在时序上的结果，比如说完这个词后的下个词的结果。
4. decoder的weight输入：用来标记target中的非padding的位置，即实际句子的长度，因为不是所有的句子的长度都一样，在实际输入的过程中，各个句子的长度都会被用统一的标示符来填充（padding）至最大长度，weight用来标记实际词汇的位置，代表这个位置将会有梯度值回传。

```
with tf.name_scope('placeholder_encoder'):
    self.encoderInputs = [tf.placeholder(tf.int32, [None, ]) for _ in
                          range(self.args.maxLengthEnco)]  # Batch size * sequence length * input dim

with tf.name_scope('placeholder_decoder'):
    self.decoderInputs = [tf.placeholder(tf.int32, [None, ], name='inputs') for _ in
                          range(self.args.maxLengthDeco)]  # Same sentence length for input and output (Right ?)
    self.decoderTargets = [tf.placeholder(tf.int32, [None, ], name='targets') for _ in
                           range(self.args.maxLengthDeco)]
    self.decoderWeights = [tf.placeholder(tf.float32, [None, ], name='weights') for _ in
                           range(self.args.maxLengthDeco)]
```

其实，数据获取后的Batch的过程可以由Tensorflow标准的batch方法来实现，而这个项目自己Batch各个输入值，因此对于初学者来说，可以观察输入值的构造，对入门RNN还是很有帮助的

#### 2.3 封装Embedding seq2seq模型

Tensorflow把常用的seq2seq模型都封装好了，比如embedding_rnn_seq2seq，这是seq2seq一个最简单的模型，一般的文本任务都会加上attention机制，但这里都用的短句子，所以attention并考虑，若想加上attention也很简单，直接修改模型名字为embedding_attention_seq2seq即可，这种封装虽然使用起来很方便，但是对于用户来说就是个黑匣子，想要自己去实现一些功能，还得去看代码。

```
decoderOutputs, states = tf.contrib.legacy_seq2seq.embedding_rnn_seq2seq(
    self.encoderInputs,  # List<[batch=?, inputDim=1]>, list of size args.maxLength
    self.decoderInputs,  # For training, we force the correct output (feed_previous=False)
    encoDecoCell,
    self.textData.getVocabularySize(),
    self.textData.getVocabularySize(),  # Both encoder and decoder have the same number of class
    embedding_size=self.args.embeddingSize,  # Dimension of each word
    output_projection=outputProjection.getWeights() if outputProjection else None,
    feed_previous=True if bool(self.args.test) and not bool(self.args.beam_search) else False

    # When we test (self.args.test), we use previous output as next input (feed_previous)
)
```
RNN输出一个句子的过程，其实是对句子里的每一个词来做整个词汇表的softmax分类，取概率最大的词作为当前位置的输出词，但是若词汇表很大，计算量会很大，那么通常的解决方法是在词汇表里做一个下采样，采样的个数通常小于词汇表，例如词汇表有50000个，经过采样后得到4096个样本集，样本集里包含1个正样本（正确分类）和4095个负样本，然后对这4096个样本进行softmax计算作为原来词汇表的一种样本估计，这样计算量会小不少。

在这里，具体的操作是定义一个全映射outputProjection对象，把隐藏层的输出映射到整个词汇表，这种映射需要参数w和b，也就是out＝w＊h＋b，h是隐藏层的输出，out是整个词汇表的输出，可以理解为一个普通的全连接层。假设隐藏层的输出是512，那么w的shape就为50000＊512，我们采样词汇表的操作可以看作是对w和b参数的采样，也就是采样出来的w为4096＊512，用这个w带入上式计算，能得出4096个输出，然后计算softmax loss，这个sampled softmax loss是原词汇表softmax loss的一种近似。outputProjection的定义请看model.py。

```

outputProjection = ProjectionOp(
    (self.textData.getVocabularySize(), self.args.hiddenSize),
    scope='softmax_projection',
    dtype=self.dtype
)

def sampledSoftmax(labels, inputs):
    labels = tf.reshape(labels, [-1, 1])  # Add one dimension (nb of true classes, here 1)

    # We need to compute the sampled_softmax_loss using 32bit floats to
    # avoid numerical instabilities.
    localWt = tf.cast(outputProjection.W_t, tf.float32)
    localB = tf.cast(outputProjection.b, tf.float32)
    localInputs = tf.cast(inputs, tf.float32)

    return tf.cast(
        tf.nn.sampled_softmax_loss(
            localWt,  # Should have shape [num_classes, dim]
            localB,
            labels,
            localInputs,
            self.args.softmaxSamples,  # The number of classes to randomly sample per batch
            self.textData.getVocabularySize()),  # The number of classes
        self.dtype)
```


#### 2.4 定义损失函数和更新方法

下面定义seq2seq模型的损失函数sequence_loss，其中sequence_loss需要softmax_loss_function参数，这个参数若不指定，那么就是默认对整个词汇表的做softmax loss，若需要采样来加速计算，则要传入上面定义的sampledSoftmax方法，这个方法的返回值是TF定义的sampled_softmax_loss。更新方法采用默认参数的Adam。

```
# Finally, we define the loss function
self.lossFct = tf.contrib.legacy_seq2seq.sequence_loss(
    decoderOutputs,
    self.decoderTargets,
    self.decoderWeights,
    self.textData.getVocabularySize(),
    softmax_loss_function=sampledSoftmax if outputProjection else None  # If None, use default SoftMax
)
tf.summary.scalar('loss', self.lossFct)  # Keep track of the cost

# Initialize the optimizer
opt = tf.train.AdamOptimizer(
    learning_rate=self.args.learningRate,
    beta1=0.9,
    beta2=0.999,
    epsilon=1e-08
)
self.optOp = opt.minimize(self.lossFct)
```

### 3. Beam Search

在测试模型的时候，比如我输入问一句话“How are you？“以及一个go开始符，那么模型就开始输出第一个词的候选集，这个候选集里的每一个词都有一个概率，一般采用贪婪的思想，就取概率最大的那个词作为当前输出，然后把这个词作为预测第二个词的输入再feed进网络，如此循环，直到模型输出end结束符，那么这句话就输出完毕。

这种把上一个时刻的输出当作下一个时刻的输入的过程在TF模型中由feed_previous参数决定：在训练模型的时候，我们是知道每一时刻的正确输入和输出的，并不需要这个过程，因此`feed_previous＝False`，而只有在测试的时候，才会需要这种过程`feed_previous＝True`

而Beam Search是这种贪婪的思想的扩展，前面是选择最大的Top 1概率的词作为当前输出，而Beam Search是选择当前Top k得分的词，当然这个得分也就是概率，那么采用这种思想，对一个问题，模型最后的输出就应该有好几种回答，这些回答根据得分排序，最终选择得分最高的句子作为最终输出。相对于前面贪婪的回答，这种搜索机制能让机器人选择更好的回答。

那么在DeepQA的基础上，我们来自己实现一下Beam Search（目前DeepQA并不支持），网上有很多实现的方法，大都是自己编写decoder，比较复杂。那么本文就采用一种非常直接的方法，依照Beam Search的思想：feed in上一时刻产生的Top k答案来产生本时刻的候选答案集，然后排序本时刻的候选答案集再选择Tok k作为本时刻的最终答案，并作为下一时刻输入，如此循环。每一时刻需要记录当前选择词的id，从第一个词到最后一个词，这些词的id构成一种选择路径。最后根据参数k，得出k条路径，每条路径有一个概率得分。这种情况下，我们是手动feed in各个候选答案，因此`feed_previous=False`


```
def beamSearchPredict(self, question, questionSeq=None):

    # question为输入的句子，这里先把每个词转为id，再加上padding和go标示符构成一个batch，这个batch就包含一个句子
    batch = self.textData.sentence2enco(question)

    if not batch:
        return None
    if questionSeq is not None:  # If the caller want to have the real input
        questionSeq.extend(batch.encoderSeqs)

    # feedDict为TF的placeholder变量以及对应的数据
    # ops为TF的需要计算的网络图
    ops, feedDict = self.model.step(batch)

	# 定义softmax操作
    def softmax(x):
        return np.exp(x) / np.sum(np.exp(x), axis=0)
	
	# path储存搜索的路径，probs里储存每个路径对应的得分（log概率）
	# beam_size对应路径的个数，储存的位置越靠后，得分越高 
    beam_size = self.args.beam_size
    path = [[] for _ in range(beam_size)]
    probs = [[] for _ in range(beam_size)]
	
	# 计算第一个词的output
    output = self.sess.run(ops[0][0], feedDict)
    for k in range(len(path)):
	    # 计算输出的softmax概率分布
        prob = softmax(output[-1].reshape(-1, ))
        # 用对数表示这些概率分布
        log_probs = np.log(np.clip(a=prob, a_min=1e-5, a_max=1))
        
        # 根据概率大小排序，取前Top－beam_size的词，记录log概率和词的id
        top_k_indexs = np.argsort(log_probs)[-beam_size:]
        path[k].extend([top_k_indexs[k]])
        probs[k].extend([log_probs[top_k_indexs[k]]])
	
	# 计算第二个词到最后一个词
    for i in range(2, self.args.maxLengthDeco):
        tmp = []
        for k in range(len(path)):
            for j in range(len(path[k])):
	            # feedDict种加入前面的词的id数据
                feedDict[self.model.decoderInputs[j + 1]] = [path[k][j]]
            # 输入feedDict至网络图，计算当前句子的output
            output = self.sess.run(ops[0][0:i], feedDict)
            # 取最后一个output（当前词的输出）做softmax和log
            prob = softmax(output[-1].reshape(-1, ))
            log_probs = np.log(np.clip(a=prob, a_min=1e-5, a_max=1))
            # 计算概率得分：P(a,b)=P(a|b)*P(b)
            # a为当前要选择的词，b为上一个已选择的词
            # Log表示： log P(a,b)=log P(a|b)+ log P(b)
            tmp.extend(list(log_probs + probs[k][-1]))
		
		# 假设上一个词的候选集有三个元素a,b,c
	    # 那么分别把这个三个元素作为输入feed进网络里，会得出三个output的结果
	    # 将这些output的概率串联到一个tmp里排序，依然取出前Top-beam_size的词作为当前的词的候选集
        top_k_indexs = np.argsort(tmp)[-beam_size:]
        indexs = top_k_indexs % self.textData.getVocabularySize()
		
		# 记录当前选择词的id和log概率得分
        for k in range(len(path)):
            probs[k].extend([tmp[top_k_indexs[k]]])
            path[k].extend([indexs[k]])
    return path

```

还有部分细节没有展出，详细请看[chatbot.py](https://github.com/PENGZhaoqing/DeepQA/blob/master/chatbot/chatbot.py)

### 4.运行

使用`python main`来训练模型，使用`python main --test interactive --beam-search --beam-size 3`来实时测试对话，下面是loss的图，混乱度为以loss为自变量的e的指数函数

![这里写图片描述](http://img.blog.csdn.net/20170907203959052?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcHBwODMwMDg4NQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

使用`python main --test`来生成对话，在save/model/model_predictions.txt中查看结果

### 5. 结果

deepQA是基于python 3.5的项目，用python 2.7也可以跑，但是需要稍微修改一些地方，另外项目还提供了训练好的参数，不想训练的可以直接导入训练好的模型。但是那些模型都是用python 3.5训练的，如果用python 2.7会导入不了这些预训练的模型。我用的是python 2.7，因此只能自己训练，由于显存的限制，训练的参数很小，效果不太好，这里给出官方的对话例子：

```
Q: Hi
A: Hi.

Q: What is your name ?
A: Laura.

Q: What does that mean ?
A: I dunno.

Q: How old are you ?
A: thirty-five.

Q: Will Google hire me ?
A: No.

Q: Tell me the alphabet
A: Fuck you.

Q: That's not nice
A: Yeah.
```

下面给出部分我用Beam Search做出来结果，beam_size=3，因此每个问题有三个答案，得分从高到低：


```
Q: Hi
A: Hi,
A: Hey, man,
A: Hello ... alright! what suicide!.?

Q: Luke, I am your father!
A: What?
A: Who?
A: We're!?!

Q: Are you ready ?
A: I'm
A: What?
A: Who? ready?

Q: When are you ready ?
A: Tomorrow.
A: Thursday.
A: I, uh., is

Q: How old are you ?
A: Eighteen.
A: Twenty.
A: I'm thirty-four.

Q: How is Laura ?
A: Fine.
A: Tolerable well
A: Good...
```

### 6. 分析

通过Beam search，有的问题能够举例多种回答，但是有的问题的后面的回答直接就失败了。有以下方面可以还可以提高：

1. 更大的训练集：训练效果强烈依赖于训练集的质量，甚至可以把几个数据集在同一个RNN网络上训练，猜想可能效果会更好
2. 一些参数的调整：如词频过滤的大小，词汇表大小，word embedding的维度大小，RNN的stack层数，输入输出句子最大长度，softmax的采样的大小，以及导入预训练的word2vec来加速训练过程等
3. 增加attention机制，对于长句子
4. 输入多个问题，只给出一个答案：这样能让网络稍微记住当前答案与前几个问题有关，目前是当前答案只与当前问题有关，问题与问题之间相互独立
5. 更高级一点，增加记忆网络，显性将特征记忆到外部储存器，然后在记忆中搜索答案返回，这样可以使记忆长期保留，是目前比较火的研究方向
6. 再高级就是增加知识图谱和一些启发函数，让模型自己去外部获取信息处理，然后返回答案，类似你问一个问题，即使机器人不知道，但是它可以去上网查资料然后返回你答案，这种level是最高级的。
