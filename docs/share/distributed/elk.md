# ElK

## 1. Elasticsearch

## 2. Logstash

## 3. Kibana

```bash
POST /_analyze
{
  "text": "我的苹果手机和你的安卓手机，我们都有身份证",
  "analyzer": "ik_smart"
}

DELETE /mytest/

PUT mytest
{
    "mappings": {
        "properties": {
            "id": {
                "type": "long"
            },
            "title": {
                "type": "text",
                "analyzer": "ik_max_word"
            }
        }
    }
}

POST /mytest/_doc
{
  "title": "我的苹果手机和你的安卓手机，我们都有身份证",
  "images": "http://image.leyou.com/12479122.jpg",
  "price": 598
}

POST /mytest/_doc
{
  "title": "工作流中的UEL表达式的使用",
  "images": "http://image.222.jpg",
  "price": 128
}

get mytest/_search
{
   "query": {
    "match_all": {
      
    }
   },
   "sort": [
        { "price": "asc" }
    ]
}　


get mytest/_search
{
  "query": { "match": { "title": "我们的身份" } }
}

```

## 4. Filebeat