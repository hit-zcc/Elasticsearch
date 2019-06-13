# 引言

在使用Elasticsearch场景中，我们或多或少会遇到嵌套文档的组合形式，在ES中称为父子文档。
父子文档的实现，至少包含以下两种方式：
1）父子文档
6.X+的父子索引的实现是Join。
2)Nested嵌套类型
 nested 类型是一种对象类型的特殊版本，它允许索引对象数组，独立地索引每个对象。

在实际的生产中存在这样的需求:一本书包含多个章节,在进行对于书籍的搜索时，需要将这本书的所有章节的得分的总分作为这本书的分值参与排序，以此能使得搜索结果更加准确，这里便需要嵌套文档的支持。所以我们将书籍作为父节点，子节点对于每一个章节并使用nested嵌套类型进行索引，大体如下
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
    "has_child":{
             "type" : "sub",
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
