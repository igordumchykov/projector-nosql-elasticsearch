# projector-nosql-elasticsearch
Creates index `words`
Uploads [data](data.json) to elasticsearch index `words`
Configures autocomplete:
1. Using custom `edge_ngram` filter and custom analyzer (to leverage 3 typos)
2. Using mapping for name field with type completion (to leverage at most 2 typos) and elasticsearch suggester

# Project Structure
- [docker compose](./docker-compose.yml) with latest elasticsearch image
- [data](./data.json) with 370105 english words

# Project Setup
## 1. Startup elasticsearch in docker
```shell
docker-compose up -d
```

## 2. Autocompletion with ngram
### 2.1 Create mapping, filter and analyzer

```shell
curl --location --request PUT 'localhost:9200/words' \
--header 'Content-Type: application/json' \
--data '{
  "{
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

### 2.2 Upload data file

```shell
curl -s -X POST "localhost:9200/_bulk" -H 'Content-Type: application/x-ndjson' --data-binary @data.json > /dev/null 2>&1
```

### 2.3 Test autocompletion

To test 2 typos, let's run a fuzzy query API for searching word `accomodate`:
1. `accomodate` - no typos
2. `acomodate` - 1 typo
3. `acomdate` - 2 typo
4. `acomdte` - 3 typos

```shell
curl --location --request GET 'localhost:9200/words/_search' \
--header 'Content-Type: application/json' \
--data '{
    "size": 100,
    "query": {
        "match": {
            "name": {
                "query": "acomte",
                "analyzer": "autocomplete"
            }
        }
    }
}'
```

Result for 3 typos (part of response)

```
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
      "value": 6344,
      "relation": "eq"
    },
    "max_score": 9.745715,
    "hits": [
      {
        "_index": "words",
        "_id": "1653",
        "_score": 9.745715,
        "_source": {
          "name": "accomodate"
        }
      },
      ...
      {
        "_index": "words",
        "_id": "1653",
        "_score": 4.4245696,
        "_source": {
          "name": "accomodate"
        }
      },
      {
        "_index": "words",
        "_id": "2613",
        "_score": 4.4245696,
        "_source": {
          "name": "acoelomate"
        }
      }
      ...
    ]
  }
}
```

## 3.Autocompletion with elasticsearch suggester and custom mapping

### 3.1 Create mapping

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
### 3.2 Upload data file

```shell
curl -s -X POST "localhost:9200/_bulk" -H 'Content-Type: application/x-ndjson' --data-binary @data.json > /dev/null 2>&1
```

To test 2 typos, let's run a fuzzy query API for searching word `accomodate`:
1. `accomodate` - no typos
2. `acomodate` - 1 typo
3. `acomodte` - 2 typos

### 3.3 Call query API

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

Result for 2 typos

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