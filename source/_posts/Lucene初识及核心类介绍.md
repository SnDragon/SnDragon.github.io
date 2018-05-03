---
title: Lucene初识及核心类介绍
date: 2018-05-03 21:39:54
tags: ["Java","搜索","Lucene"]
---
[Lucene](http://lucene.apache.org)是目前最流行的Java开源搜索引擎类库,最新版本为7.3.0。Lucene通常用于全文检索,Lucene具有简单高效跨平台等特点,因此有不少搜索引擎都是基于Lucene构建的,例如:[Elasticsearch](http://www.elastic.co/products/elasticsearch),[Solr](http://lucene.apache.org/solr/)等等。了解Lucene可以让我们更好地理解全文检索的内部原理,本篇文章将简单介绍Lucene及其核心类。
## Lucene简介
现代搜索引擎的两大核心就是索引和搜索,建立索引的过程就是对源数据进行处理,例如过滤掉一些特殊字符/词语(如虚词等),单词大小写转换,分词,建立倒排索引等以支持后续高效准确的搜索。而搜索则是直接提供给用户的功能，尽管面向的用户不同,诸如百度、谷歌等互联网公司以及各种企业都提供了各自的搜索引擎。搜索过程需要对搜索关键词进行分词等处理,然后在引擎内部构建查询,还要根据相关度对搜索结果进行排名,最终把命中结果展示给用户。

Lucene只是一个提供索引和搜索的类库,并不是一个应用,程序员需要根据自己的业务场景进行如数据获取、数据预处理、用户界面提供等工作。

下图为Lucene与应用程序的关系:

![搜索应用程序和 Lucene 之间的关系](https://www.ibm.com/developerworks/cn/java/j-lo-lucene1/fig001.jpg)


## Lucene Demo
接下来先看一个简单的Demo:

引入Maven依赖:
```xml
  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
    <lucene.version>7.3.0</lucene.version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.11</version>
      <scope>test</scope>
    </dependency>

    <dependency>
      <groupId>org.apache.lucene</groupId>
      <artifactId>lucene-core</artifactId>
      <version>${lucene.version}</version>
    </dependency>

    <dependency>
      <groupId>org.apache.lucene</groupId>
      <artifactId>lucene-queryparser</artifactId>
      <version>${lucene.version}</version>
    </dependency>

  </dependencies>
```

索引类:
```java
import org.apache.lucene.analysis.*;
import org.apache.lucene.analysis.standard.*;
import org.apache.lucene.document.*;
import org.apache.lucene.index.*;
import org.apache.lucene.store.*;

import java.io.*;
import java.nio.charset.*;
import java.nio.file.*;
import java.nio.file.attribute.*;

public class IndexFiles {
    public static void main(String[] args) {
        String indexPath = "D:/lucenedata/index";    //建立索引文件的目录
        String docsPath = "D:/lucenedata/docs";     //读取文本文件的目录

        Path docDir = Paths.get(docsPath);

        IndexWriter writer = null;
        try {
            //存储索引数据的目录
            Directory dir = FSDirectory.open(Paths.get(indexPath));
            //创建分析器
            Analyzer analyzer = new StandardAnalyzer();
            IndexWriterConfig iwc = new IndexWriterConfig(analyzer);
            iwc.setOpenMode(IndexWriterConfig.OpenMode.CREATE);

            writer = new IndexWriter(dir, iwc);
            indexDocs(writer, docDir);

            writer.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private static void indexDocs(final IndexWriter writer, Path path) throws IOException {
        if (Files.isDirectory(path)) {
            Files.walkFileTree(path, new SimpleFileVisitor<Path>() {
                @Override
                public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) {
                    try {
                        indexDoc(writer, file);
                    } catch (IOException ignore) {
                        //不索引那些不能读取的文件,忽略该异常
                    }
                    return FileVisitResult.CONTINUE;
                }
            });
        } else {
            indexDoc(writer, path);
        }
    }

    private static void indexDoc(IndexWriter writer, Path file) throws IOException {
        try (InputStream stream = Files.newInputStream(file)) {
            //创建一个新的空文档
            Document doc = new Document();
            //添加字段
            Field pathField = new StringField("path", file.toString(), Field.Store.YES);
            doc.add(pathField);
            Field contentsField = new TextField("contents", new BufferedReader(new InputStreamReader(stream, StandardCharsets.UTF_8)));
            doc.add(contentsField);
            System.out.println("adding " + file);
            //写文档
            writer.addDocument(doc);
        }
    }
}
```

查找类:
```java
import org.apache.lucene.analysis.*;
import org.apache.lucene.analysis.standard.*;
import org.apache.lucene.document.*;
import org.apache.lucene.index.*;
import org.apache.lucene.queryparser.classic.*;
import org.apache.lucene.search.*;
import org.apache.lucene.store.*;

import java.io.*;
import java.nio.charset.*;
import java.nio.file.*;

public class SearchFiles {
    public static void main(String[] args) throws Exception{
        String indexPath = "D:/lucenedata/index";    //建立索引文件的目录
        String field = "contents";
        IndexReader reader = DirectoryReader.open(FSDirectory.open(Paths.get(indexPath)));
        IndexSearcher searcher = new IndexSearcher(reader);
        Analyzer analyzer = new StandardAnalyzer();

        BufferedReader in = null;
        in = new BufferedReader(new InputStreamReader(System.in, StandardCharsets.UTF_8));
        QueryParser parser = new QueryParser(field, analyzer);
        System.out.println("Enter query:");
        String line = in.readLine();
        if (line == null || line.length() == -1) {
            return;
        }
        line = line.trim();
        if (line.length() == 0) {
            return;
        }

        Query query= parser.parse(line);
        System.out.println("Searching for:"+query.toString(field));
        doPagingSearch(searcher,query);
        in.close();
        reader.close();
    }

    private static void doPagingSearch(IndexSearcher searcher, Query query) throws IOException {
        //TopDocs保存搜索结果
        TopDocs results = searcher.search(query, 10);
        ScoreDoc[] hits = results.scoreDocs;
        int numTotalHits = Math.toIntExact(results.totalHits);
        System.out.println(numTotalHits + " total matching documents");
        for(ScoreDoc hit:hits){
            Document document=searcher.doc(hit.doc);
            System.out.println("文档:"+document.get("path"));
            System.out.println("相关度:"+hit.score);
            System.out.println("================================");
        }

    }
}
```
## Lucene核心类介绍
### 索引过程核心类
正如在IndexFiles类所看到的,执行简单的索引过程需要用到以下几个类:
* IndexWriter
* Director
* Analyzer
* Document
* Field

![索引过程](https://upload-images.jianshu.io/upload_images/4778432-8204e3e27f68ee7a.png)

#### IndexWriter
IndexWriter是索引过程的核心组件,主要负责创建和管理索引的,但不能用于读取或搜索索引。IndexWriter需要开辟一定空间来存储索引,该功能由Directory完成。

#### Directory
Directory类描述了Lucene索引的存放位置。它是一个抽象类,其子类负责具体制定索引的存储路径。Directory的主要实现有FSDirectory、RAMDirectory等。

IndexWriter不能直接索引文本,需要Analyzer先把文本分割成语汇单元。

#### Analyzer
文本文件在被索引之前,需要经过Analyzer(分析器)处理。Analyzer负责从被索引文本文件中提取语汇单元。同Directory一样,Analyzer是一个抽象类,Lucene内置也有一些实现类。有的用于跳过stop words(指一些常用的且不能帮助区分文档的词,如a、an、the等);有的用于将语汇单元转换为小写形式以使搜索过程能忽略大小写差别。

分析器的分析对象为文档(Document)的集合,可以理解为RDBMS中的Table。

#### Document
Document(文档)对象代表一些域(Field)

#### Field
Field对应Document的一个字段,例如一篇文章有标题、作者、正文等字段。

### 搜索过程核心类
正如在SearchFiles看到的,简单的搜索过程需要用到以下类:
* IndexSearcher 
* Analyzer
* QueryParser
* Query
* TopDocs

#### IndexSearcher
执行搜索的核心类,IndexSearcher类用于搜索由IndexWriter类创建的索引,这个类提供了许多搜索方法

#### QueryParser
查询分析器,根据指定的输入和分析器构建相应的查询器。

#### Query
查询对象,通过IndexSearcher#search调用。Query是一个抽象类,实现类有TermQuery、BooleanQuery、PhraseQuery等

#### TopDocs
TopDocs类是一个简单的指针容器,指针一般指向前N个排名的搜索结果,搜索结果即匹配条件的文档。TopDocs会记录前N个结果中每个结果的int docID和浮点数型分数(反映相关度)

### 使用Lucene核心步骤
要使用Lucene,有以下4个核心步骤:
1. 通过增加Fields创建Documents
2. 创建一个IndexWriter并通过addDocument()添加文档
3. 调用*QueryParser.parse()*构建查询
4. 创建一个IndexSearcher并将查询对象传递给其search()方法