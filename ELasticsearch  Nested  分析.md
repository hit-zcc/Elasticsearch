# 引言

## 前言
在使用Elasticsearch场景中，我们或多或少会遇到嵌套文档的组合形式，在ES中称为父子文档。
父子文档的实现，至少包含以下两种方式：
 
 1）父子文档
 
 6.X+的父子索引的实现是Join。
 
 2)Nested嵌套类型
 
 nested 类型是一种对象类型的特殊版本，它允许索引对象数组，独立地索引每个对象。
 
 ## 目前存在的问题
 
 现在我们的电子书是将一本电子书按章节进行切分，并且每个章节作为一个doc放入索引中，所以在进行搜索时是按照一个章节的匹配度来进行排序的，造成的后果就是虽然返回的章节有很高的分数，但是对于这本书来说其实并不是很好。

所以改进的措施如下:

一本书包含多个章节,在进行对于书籍的搜索时，需要将这本书的所有章节的得分的总分作为这本书的分值参与排序，以此能使得搜索结果更加准确，这里便需要嵌套文档的支持。所以我们将书籍作为父节点，子节点是每一个章节并使用nested嵌套类型进行索引，大体如下
```
{
  "mapping": {
    "_doc": {
      "dynamic": "false",
      "_all": {
        "enabled": false
      },
      "properties": {
        "chapters": {
          "type": "nested",
          "properties": {
            "chapter": {
              "type": "text",
              "term_vector": "with_positions_offsets",
              "analyzer": "index",
              "search_analyzer": "search"
            }
          }
        }
      }
    }
  }
}
```

然后使用has_child 配合 innerhit 进行查询，主要为了能够返回一本书中前几个最有结果所以加了innerhit，并使用子文档的所有得分作为父文档的得分，大体如下
```
{
  "query":{
    "match":{
    ”title“:"谈"
    },
    "nested": {
            "path": "chapter"
            "score_mode" : "sum",
             "query": {
              "bool": {
                "must": {
                  "bool": {
                    "should": [
                      {
                        "match_phrase": {
                          "chapter": {
                            "query": "幸福",
                            "analyzer": "search_grams",
                            "slop": 20,
                            "boost": 2
                          }
                        }
                      }
                    ]
                  }
                }
              }
            },
             "inner_hits": {
              "name": "test_name",
              "from": 0,
              "size": 2,
              "highlight": {
                "pre_tags": "<hl>",
                "post_tags": "</hl>",
                "order": "score",
                "number_of_fragments": 1,
                "fragment_size": 120,
                "fields": {
                  "chapter": {
                    "type": "experimental",
                    "options": {
                      "return_snippets_and_offsets": true
                    }
                  }
                }
              }
            }
    }
  }
}
```

但在使用该查询时，速度非常慢，需要2000毫秒左右才能返回，下面通过源码追查其慢的原因。

# ES Nested 源码解析
## query阶段
1.在NestedQueryBuilder.java中其在构造nested查询的时候，其会新建一个parentFilter,该实例用以获得所有父文档docid,并在后面检查nest中是否嵌套nest（此处先不讨论）,最终使用innerQuery和parentFilter等构造出一个ESToParentBlockJoinQuery。
```
        final BitSetProducer parentFilter;
        Query innerQuery;
        ObjectMapper objectMapper = context.nestedScope().getObjectMapper();
        if (objectMapper == null) {
            parentFilter = context.bitsetFilter(Queries.newNonNestedFilter(context.indexVersionCreated()));
        } else {
            parentFilter = context.bitsetFilter(objectMapper.nestedTypeFilter());
        }

        try {
            context.nestedScope().nextLevel(nestedObjectMapper);
            innerQuery = this.query.toQuery(context);
        } finally {
            context.nestedScope().previousLevel();
        }

        // ToParentBlockJoinQuery requires that the inner query only matches documents
        // in its child space
       if (new NestedHelper(context.getMapperService()).mightMatchNonNestedDocs(innerQuery, path)) {
            innerQuery = Queries.filtered(innerQuery, nestedObjectMapper.nestedTypeFilter());
        }

        return new ESToParentBlockJoinQuery(innerQuery, parentFilter, scoreMode,
                objectMapper == null ? null : objectMapper.fullPath());

```
2.在ESToParentBlockJoinQuery中会初始化一个ToParentBlockJoinQuery，在ToParentBlockJoinQuery中会通过DocValuesFieldExistsQuery[field=_primary_term]  得到父空间下的文档（父文档会默认增加这个特殊字段），并同时生成一个 lucene的查询语法  +(+title:谈 ToParentBlockJoinQuery ((chapters.chapter:幸福)^8.0))   以此来获取子父空间中的所有文档。
 
```
      final BitSet parents = parentsFilter.getBitSet(context);
      if (parents == null) {
      return new ScorerSupplier() {    
        public Scorer get(long leadCost) throws IOException {
          return new BlockJoinScorer(BlockJoinWeight.this, childScorerSupplier.get(leadCost), parents, scoreMode);
        }
      };
      
     
 ``` 
 3.最终在BlockJoinScorer得出最终一篇父文档的得分和父文档的doc id，其中childApproximation和parentApproximation分别代表子文档和父文档匹配结果的迭代器
  
  注：在索引的迭代器中 ，这里面既有nested的文档，也有parent的文档，nested文档和parent文档存储在一起并且只有parent文档会又source，举个例子，doc1 有三个nested，那么他们的id排序就是1,2,3,4（parent); doc2 有2个nested， 那么他们排序就是 5，6，7（parent）
所以存在下面这个逻辑
    查看匹配的子文档id是否小于父文档的id，如果是则说明该子文档就是属于该父文档，则会对该子文档进行打分并且根据scoreMode 将分值赋予父文档。
 ```   
   if (childApproximation.docID() >= parentApproximation.docID()) {
        return;
      }
      double score = scoreMode == ScoreMode.None ? 0 : childScorer.score();
      int freq = 1;
      while (childApproximation.nextDoc() < parentApproximation.docID()) {
        if (childTwoPhase == null || childTwoPhase.matches()) {
          final float childScore = childScorer.score();
          freq += 1;
          switch (scoreMode) {
            case Total:
            case Avg:
              score += childScore;
              break;
            case Min:
              score = Math.min(score, childScore);
              break;
            case Max:
              score = Math.max(score, childScore);
              break;
            case None:
              break;
            default:
              throw new AssertionError();
          }
        }
      }
      if (scoreMode == ScoreMode.Avg) {
        score /= freq;
      }
      this.score = (float) score;
      this.freq = freq;
    }
``` 
   4.在进行完成这些操作后就完成了父节点的匹配从而返回的匹配的父节点的文档id 
  
  总结，上面的流程经过试验速度在几十毫秒之内完成，速度很快
  
  ## fetch阶段
     
     
   1.下面是fetch的常规操作
   
   在fetch阶段首先拿到所有shard返回的docid和score，然后进行重排序，拿到真正的TopN的文档id，然后从每个shard中fetch真正的文档，但是在从索引中fetch真正文档的时候有些问题,其会根据每个父节点的id取出_source（注意这里取出source会将存储的source全部取出，所以虽然父文档不需要返回大字段chapters，其仍会取出，只不过在填充过后将其丢弃），将其填充到要返回的字段中，即_fileds，这里的问题就是一个我们一本书包含很多章节，每个章节又很长，这里他是把整本书内容给拉取了出来。
     
  ``` 
   loadStoredFields(context, subReaderContext, fieldsVisitor, subDocId);
  ``` 
    
    
   2.其实如果没有inner hit 这里就结束了，但有了innerhit这里性能会更慢，因为在query阶段只返回了父文档的得分，而在innerhit中的size为2就是说要返回子文档中的top2，这个在第一次query中虽然已经计算过一次但是并没有传递下来，所以这里会重新将匹配的子文档的得分在计算一遍
    
 ``` 
    InnerHitsContext.InnerHitSubContext innerHits = entry.getValue();
    TopDocs[] topDocs = innerHits.topDocs(hits);   //这里重新计算了子文档的得分并只保留topn
 ```
 
  3.在计算完能够获得到子文档的id，所以需要在对子id进行一次fetch，将子id需要的source填充到子doc中，,由于子文档是没有_source的 ，这里的_source会拉取整个父文档的_source,所以这里又进行了一次大字段chapters的拉取，最终通过填充到子文档匹配结果中chapter做高亮处理。
     
 ``` 
      FieldsVisitor rootFieldsVisitor = new FieldsVisitor(needSource);
      loadStoredFields(context, subReaderContext, rootFieldsVisitor, rootSubDocId);
 ```
     
   总结一下，上面的流程由于存在两次拉取chapters过程，而chapters存储的是整本书的文本内容，所以会耗费巨大的时间，为了更加直观
   举个栗子  按平均按每本书50个章节，查询结果需要返回20本书的top5个章节来算的话  他其实拉取的chapter达到了50*20*2=2000条  这必然是巨慢的
  
  
 #  解决方案

  1.不存储大文本的source，这样在拉取source时就不会存在慢的情况，但带来的弊端就是没法得到高亮的文档，高亮只会返回偏移，这就需要后端对偏移进行处理
    
  2.使用join类型替代nest，join类型的实现方式和nest不同，其每个文档都会又自己的source，只不过会多存储反应子父文档关系的文档，但他也避免了nest存在的问题
    
  3.如果不使用ScoreMode.sum 的话查询结果会快一倍，主要是因为如果使用章节的总分作为书的分，则返回的的书都是章节巨多的那种，所以拉取的数据就大，而使用其他的的ScoreMode方式就返回很多章节数小的书所以返回速度会快，这也正说明了上述的理论
  
  # 结语
  
  **nested 嵌套适合子文本较小，父子结构较多的场景。**
    


     
