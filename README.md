# R-TextClassification

##用R语言做文本分类

###一、R语言
因项目需要，结合自身专业知识，接触了R语言及一些常用分类器。主要是用R语言来做基本的文本分类这个任务，在此记录下自己曾经的学习历程。

R语言与起源于贝尔实验室的s语言类似，R也是一种为统计计算和绘图而生的语言和环境，它是一套开源的数据分析解决方案，由一个庞大且活跃的全球性研究型社区维护。

另外，R语言还是一门脚本语言，在绘图方面有着非常强的能力，它可以让你集中到你要设计的逻辑上来，而不必太过纠结于代码的实现。它的包实在太丰富，几乎能满足你全部的需要。我使用的IDE是RStudio。然后介绍几个我在文本分类里用到的包：

    RODBC 连接数据库的包，我主要用它读取数据库里的信息然后保存到本地，制成文本文件。
    tm 文本挖掘的包，对数据的读入、输出及语料库的提取、转化、过滤等等，最终转化成文档-词条矩阵。
    RwordSeg 这是一个分词包，需配合rJava包一起使用。分词效果对我而言已够用。
    maxent 最大熵分类器包，选择最大熵分类器是因为它效率较高。
    e1071 该包中有svm分类器。具体可看使用者文档。

为了激发兴趣，可以学习其中一个很有意思的包：词云包——WordCloud
代码如下：

    library(wordcloud)  #加载wordcloud包
    library(RColorBrewer) #加载颜色包
    png(file="WordCloud.png", bg="white",width = 600, height = 780) #新建一个png的文件作为词云文件。
    
    colors = brewer.pal(8,"Dark2")[-(1:4)]
    data = read.csv("wordcount.txt") #读取设置的词及频度，用于显示。
    
    #然后调用wordcloud函数，每个参数都有各自的含义，具体可在网上查阅。
    wordcloud(data$name,data$count,scale=c(3,0.4),min.freq = -Inf,max.words=178,colors =             colors,random.order = F,random.color = T,ordered.colors = F)
    
    dev.off() 



截个图
wordcloud.txt:

![image](https://github.com/txHe/R-TextClassification/blob/master/wordcloudtxt.PNG)

wordcloud.png:

![image](https://github.com/txHe/R-TextClassification/blob/master/WordCloud.png)

###二、制作语料库
我的分类很简单，就是给你一段文字，然后将它分类到指定的类别。当然，这是前提是需要大量的语料库，且已经分好类，即有监督的分类方法。不过我的类别较多，不是二元分类，但是目前的分类器都是二元的，二元的当然可以改造成多元分类器。有One to One 和 One to the other分类。但是，将二元分类器改造成多元分类，在效率上很成问题，尤其SVM，比较慢，影响项目的实际使用。在稍一考虑后，决定使用最大熵分类器。所幸，R语言里的maxent包，已集成最大熵分类器，且它会根据类别的相似度打分，可以得到该文档在各类别下相似度的分数，得到排名，从而得出与之相近的多个类别。无疑是非常有用的。

1、文本数据在经过一些处理后格式就是：
<table>
<tbody>
<tr><td><em>文本标题(Title)</em></td><td><em>文本描述（Description)</em></td><td><em>类别(Type)</em></td></tr>
<tr><td>……</td><td>……</td><td>A</td></tr>
<tr><td>……</td><td>……</td><td>B</td></tr>
<tr><td>……</td><td>……</td><td>C</td></tr>
</tbody>
</table>

为简单处理，直接以词类作为特征，暂时将标题也作为特征。所以先将标题和描述合并。
可通过如下函数：

    BindData <- function(data)
    {
      #将标题和描述合并为一个表
      temp <- data.frame(des=NULL,typ=NULL)
      #x <- NULL
      for(i in 1:dim(data)[1])
      {
        x <- paste(data[i,1],data[i,2],sep=",") #合并标题和描述
        tempx <- data.frame(des=x) #创建临时数据框用于合并
        
        y <- as.character(data[i,3])
        tempy <- data.frame(typ=y)
        
        tempxy <- cbind(tempx,tempy)
        temp <- rbind(temp,tempxy)
        print(i)
      }
      return (temp)
    }
完成后：

<table>
<tbody>
<tr><td><em>文本(Des)</em></td><td><em>类别(Type)</em></td></tr>
<tr><td>……</td><td>A</td></tr>
<tr><td>……</td><td>B</td></tr>
<tr><td>……</td><td>C</td></tr>
</tbody>
</table>

可以算作“初始”语料库。

**2、继续制作语料库。**

主要步骤（采用比较简单的步骤）：

 **1. 分词**

 **2. 去停用词**

 **3. 生成文档-词项矩阵(tf-idf matrix)**

代码如下：

    library(tm)
    library(rJava)
    library(Rwordseg)
    #读取刚刚的“初始”语料库。
    text= read.table("maxent_10.txt",header = TRUE,row.names=1,fileEncoding = "UTF-8")
    #读取停用词，stopwords_CN可在网上下载到。
    data_stw <- readLines("stopwords_CN.txt",encoding  = "UTF-8")
    
    #去停用词的函数，将分词后的词项，如有符合停用词的，则去掉。这个函数在后面讲会调用
    removeStopWords<-function(x,words){
      ret<-character(0)
      index<-1
      it_max<-length(x)
      while(index<=it_max){
        if(length(words[words==x[index]])<1)ret<-c(ret,x[index])
        index<-index+1
      }
      ret
    }
    
    #新建列表，用于存放去停用词后的，各文档词项。
    doc_CN <- list()
    for(j in 1:dim(text)[1])
    {
      x <- c(segmentCN(as.character(text[j,1])),nosymbol=TRUE) #对文档分词
      doc_CN[[j]] <- removeStopWords(x,data_stw)    #去停用词
    }
    
    kvid <- Corpus(VectorSource(doc_CN)) #调用tm包中的函数，生成语料库格式文档。
    
    kvid <- tm_map(kvid,stripWhitespace)#去除文档中因去停用词导致的空白词。
    
    #生成词项-文档矩阵(TDM)
    control=list(removePunctuation=T,minDocFreq=1,wordLengths = c(1, Inf)) 
    #默认的加权方式是TF-IDF,removePunctution，去除标点，
    #minDocFreq = 1表示至少词项至少出现了1次，wordLengths则表示词的长度。
    
    **有个特别说明这边也可以用DocumentTermMatrix(kvid,control)——文档-词项矩阵
    **这样就是我最适合的模型了。后面有介绍。
    tdm=TermDocumentMatrix(kvid,control)#词项-文档矩阵
    
    #length(tdm$dimnames$Terms)，用于查看当前的维度
    tdm_removed=removeSparseTerms(tdm, 0.99) #一般词类做特征，维度都非常大，故需要降维。
    #length(tdm_removed$dimnames$Terms)
    
    ###tdm_removed可以用inspect(tdm_removed[1:524,1:5])查看，1:524表示1到524行，1：5表示1到5行。
    
    #读取类别和其对应的数量。为的是在词项文档矩阵后加入类别，便于后来的分类。
    #这段代码很随意，只是针对我自己的文档来做的。
    typ_text = read.table("部门类别及数量.txt",sep='\t',header = TRUE,row.names=1,fileEncoding = "UTF-8")
    n=1
    #我的文档有10个类别，因此生成10个文本，每个文本都是文档词项矩阵和对应类别，且我是按次序排好的
    for(i in 1:10)
    {
      m <- n + typ_text[i,2]
      ts <- inspect(tdm_removed[1:length(tdm_removed$dimnames$Terms),n:m-1])
      #这是转置，先前没考虑那么多所以用了TermDocumentMatrix(kvid,control)。
      #如果用了DocumentTermMatrix(kvid,control)，就不需要转置了。
      tk <- t(ts) 
      type <- data.frame(tp=rep(typ_text[i,1],dim(tk)[1]))  
      tm <- cbind(tk,type)  #加入类别
      filename <- paste(i,'.txt',sep = "")
      write.table(tm,filename,sep = "\t", col.names = NA,fileEncoding = "UTF-8")
      n <- typ_text[i,2] + n
    }
    
到这，就生成了我所需要的语料库了。各人情况不同，可互相参考。

**文档-词项矩阵：**

<table>
<tbody>
<tr><td><em>文本编号\词项</em></td><td><em>词项1</em></td><td><em>词项2</em></td><td><em>词项3</em></td><td><em>词项4</em></td></tr>
<tr><td>001</td><td>1</td><td>1</td><td>1</td><td>1</td></tr>
<tr><td>002</td><td>1</td><td>1</td><td>1</td><td>1</td></tr>
<tr><td>003</td><td>1</td><td>1</td><td>1</td><td>1</td></tr>
</tbody>
</table>
    
**词项-文档矩阵：**


<table>
<tbody>
<tr><td><em>文本编号\词项</em></td><td><em>001</em></td><td><em>002</em></td><td><em>003</em></td><td><em>004</em></td></tr>
<tr><td>词项1</td><td>1</td><td>1</td><td>1</td><td>1</td></tr>
<tr><td>词项2</td><td>1</td><td>1</td><td>1</td><td>1</td></tr>
<tr><td>词项3</td><td>1</td><td>1</td><td>1</td><td>1</td></tr>
</tbody>
</table>

明显看出，文档-词项矩阵符合我的要求。

###三、训练、分类和计算P，R，F值
直接上代码：

    library(tm)
    library(maxent)
    
    traindata <- data.frame(NULL)
    testdata <- data.frame(NULL)
    
    #循环测试
    for(i in 1:10)
    {
      filename <- paste(i,'.txt',sep="")
      text = read.table(filename, header = TRUE, sep = "\t", row.names=1,fileEncoding = "UTF-8")
      len <- dim(text)[1] 
      sam <- trunc(len * 2 / 3) #取文档2/3的数据。trunc函数用于取整
      traindata <- rbind(traindata,text[1:sam,]) #将2/3的数据放置于训练集
      k <- sam + 1
      testdata <- rbind(testdata,text[k:len,]) #剩余的数据放置于测试集
    }
    
    #计算用于最大熵的训练时间
    ptm <- proc.time()
    model <- maxent(traindata[,1:556],traindata$tp) #训练模型，顺便计算下时间。
    ptms <- proc.time() - ptm
    print(ptms)
    
    m <- testdata[,1:556]
    n <- testdata$tp
    #计算最大熵模型用于测试的时间
    ptm <- proc.time()
    
    ms <- predict.maxent(model,m)   #测试
    
    ptms <- proc.time() - ptm
    print(ptms)
    
    #计算准确率
    kn <- as.character(n) #类别数组
    km <- ms[,1]          #预测后的类别数组

    calculate_mean <- function(kn,km)
    {
      num <- 0
      for(i in 1:length(kn))
      {
        if(kn[i]==km[i])
        {
          num <- num + 1
        }
      }
      return (num/length(kn))
    }
    
    print(calcu;ate_mean(kn,km))
    
    #################################
    计算P,R,F值部分在文件Maxent.R里。
    #################################

预测完之后，就可以获得分类的结果。这里对结果用P,R和F值进行性能的分析。

##总结
纵观全文，其实这几篇文章只能算作一个入门教程，毕竟文本分类在大项目级别的使用还是需要很多特殊的技术作为支撑的。在此，仅是希望能通过这篇文章激发R语言爱好者的一些兴趣。
