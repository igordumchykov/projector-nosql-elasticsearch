# projector-nosql-elasticsearch
Creates index `words`
Uploads [data](data.json) to elasticsearch index `words`
Configures autocomplete:
1. Using mapping for name field with type completion (to leverage at most 2 typos) and elasticsearch suggester
2. Using custom `edge_ngram` filter and custom analyzer (to leverage 2 typos)

# Project Structure
- [docker compose](./docker-compose.yml) with latest elasticsearch image
- [data](./data.json) with 370105 english words

# Project Setup
## 1. Startup elasticsearch in docker
```shell
docker-compose up -d
```
## 2.Autocompletion with elasticsearch suggester and custom mapping

### 2.1 Create mapping

```shell
curl --location --request PUT 'localhost:9200/words' \
--header 'Content-Type: application/json' \
--data '{
  "mappings": {
      "properties": {
        "name": {
          "type": "completion"
        }
    }
  }
}'
```
### 2.2 Upload data file

```shell
curl -X POST "localhost:9200/_bulk" -H 'Content-Type: application/x-ndjson' --data-binary @data.json
```

To test 2 typos, let's run a fuzzy query API for searching word `accomodate`:
1. `accomodate` - no typos
2. `acomodate` - 1 typo
3. `acomodte` - 2 typos

### 2.3 Call query API

```shell
curl --location 'localhost:9200/words/_search' \
--header 'Content-Type: application/json' \
--data '{
    "suggest": {
        "words": {
            "prefix": "acomodte",
            "completion": {
                "field": "name",
                "fuzzy": {
                    "fuzziness": "2"
                }
            }
        }
    }
}'
```

### 2.4 Result for 2 typos

```json
{
    "took": 5,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 0,
            "relation": "eq"
        },
        "max_score": null,
        "hits": []
    },
    "suggest": {
        "words": [
            {
                "text": "acomodte",
                "offset": 0,
                "length": 8,
                "options": [
                    {
                        "text": "accomodate",
                        "_index": "words",
                        "_id": "1653",
                        "_score": 2.0,
                        "_source": {
                            "name": "accomodate"
                        }
                    }
                ]
            }
        ]
    }
}
```


## 3. Autocompletion with edge_ngram
### 3.1 Create mapping, filter and analyzer

```shell
curl --location --request PUT 'localhost:9200/words' \
--header 'Content-Type: application/json' \
--data '{
  "settings": {
    "analysis": {
      "filter": {
        "autocomplete_filter": { 
          "type": "edge_ngram",
          "min_gram": 1,
          "max_gram": 20
        }
      },
      "analyzer": {
        "autocomplete": { 
          "type": "custom",
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "autocomplete_filter"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "autocomplete", 
        "search_analyzer": "standard"
      }
    }
  }
}'
```

### 3.2 Upload data file

```shell
curl -X POST "localhost:9200/_bulk" -H 'Content-Type: application/x-ndjson' --data-binary @data.json
```

### 3.3 Test autocompletion

To test 2 typos, let's run a fuzzy query API for searching word `accomodate`:
1. `accomodate` - no typos
2. `acomodate` - 1 typo
3. `acomodte` - 2 typos

```shell
curl --location --request GET 'localhost:9200/words/_search' \
--header 'Content-Type: application/json' \
--data '{
    "query": {
        "fuzzy": {
            "name": {
                "value": "acomodt",
                "fuzziness": "auto"
            }
        }
    }
}'
```

### 3.4 Result for 2 typos

```json
{
    "took": 6,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 2,
            "relation": "eq"
        },
        "max_score": 14.6822195,
        "hits": [
            {
                "_index": "words",
                "_id": "1653",
                "_score": 14.6822195,
                "_source": {
                    "name": "accomodate"
                }
            },
            {
                "_index": "words",
                "_id": "64321",
                "_score": 13.983066,
                "_source": {
                    "name": "comonte"
                }
            }
        ]
    }
}
```