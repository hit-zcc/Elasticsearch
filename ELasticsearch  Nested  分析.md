# 引言

在使用Elasticsearch场景中，我们或多或少会遇到嵌套文档的组合形式，在ES中称为父子文档。
父子文档的实现，至少包含以下两种方式：
 
 1）父子文档
 
 6.X+的父子索引的实现是Join。
 
 2)Nested嵌套类型
 
 nested 类型是一种对象类型的特殊版本，它允许索引对象数组，独立地索引每个对象。

在实际的生产中存在这样的需求:

一本书包含多个章节,在进行对于书籍的搜索时，需要将这本书的所有章节的得分的总分作为这本书的分值参与排序，以此能使得搜索结果更加准确，这里便需要嵌套文档的支持。所以我们将书籍作为父节点，子节点对于每一个章节并使用nested嵌套类型进行索引，大体如下
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

然后使用has_child 配合 innerhit进行查询，主要为了能够返回一本书中前几个最有结果所以加了innerhit，大体如下
```
{
  "query":{
    "match_all":{
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

但在使用该查询时，速度非常慢，下面通过源码追查其慢的原因。

# ES Nested 源码解析 
1.在NestedQueryBuilder.java中其在构造nested查询的时候，其会新建一个parentFilter,该实例用以产生匹配父层query的结果,并在后面检查nest中是否嵌套nest（此处先讨论）,最终使用innerQuery和parentFilter等构造出一个ESToParentBlockJoinQuery。
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
2.在ESToParentBlockJoinQuery中会初始化一个ToParentBlockJoinQuery
```
public ESToParentBlockJoinQuery(Query childQuery, BitSetProducer parentsFilter, ScoreMode scoreMode, String path) {
        this(new ToParentBlockJoinQuery(childQuery, parentsFilter, scoreMode), path);
    }
```
3，在ToParentBlockJoinQuery中会进行真正的查询，主要是先或得到父查询的结果，然后

```
      final BitSet parents = parentsFilter.getBitSet(context);
      if (parents == null) {
      return new ScorerSupplier() {    
        public Scorer get(long leadCost) throws IOException {
          return new BlockJoinScorer(BlockJoinWeight.this, childScorerSupplier.get(leadCost), parents, scoreMode);
        }
      };
      
     
 ``` 
 4.最终在BlockJoinScorer得出最终一篇付父文档的得分和父文档的doc id，其中childApproximation和parentApproximation分别代表子文档和父文档匹配结果的迭代器
  注：索引一个迭代器，这里面既有nested的文档，也有parent的文档，nested文档和parent文档存储在一起并且只有parent文档会又source，举个例子，doc1 有三个nested，那么他们的id排序就是1,2,3,4（parent); doc2 有2个nested， 那么他们排序就是 5，6，7（parent），所以存在下面这个逻辑
          
   1.查看匹配的子文档id是否和大于等于父文档的id，如果不是则说明该子文档就是属于该父文档并且满足两个条件，则会对该子文档进行打分并且根据scoreMode 将分值赋予父文档
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
