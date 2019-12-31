# ELK



There are a variety of ingest options for Elasticsearch, but in the end they all do the same thing: put JSON documents into an Elasticsearch index.



Indexing documents in bulk

```sh
curl -H "Content-Type: application/json" -XPOST "localhost:9200/bank/_bulk?pretty&refresh" --data-binary "@accounts.json"
curl "localhost:9200/_cat/indices?v"
```

Searching

The following request retrieves all documents in the `bank` index sorted by account number:

```sh
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": { "match_all": {} },
  "sort": [
    { "account_number": "asc" }
  ]
}
'

```

By default, the `hits` section of the response includes the first 10 documents that match the search criteria:

```json
{
  "took" : 7,			#how long it took Elasticsearch to run the query, in milliseconds
  "timed_out" : false,  #whether or not the search request timed out
  "_shards" : {			#how many shards were searched and a breakdown of 
    "total" : 1,		#		how many shards succeeded, failed, or were skipped.
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1000,		#how many matching documents were found
      "relation" : "eq"		
    },
    "max_score" : null,		#the score of the most relevant document found
    "hits" : [	
      {
        "_index" : "bank",
        "_type" : "_doc",
        "_id" : "0",
        "_score" : null,		# the document’s relevance score
        "_source" : {			#     (not applicable when using match_all)	
          "account_number" : 0,
          "balance" : 16623,
          "firstname" : "Bradshaw",
          "lastname" : "Mckenzie",
          "age" : 29,
          "gender" : "F",
          "address" : "244 Columbus Place",
          "employer" : "Euron",
          "email" : "bradshawmckenzie@euron.com",
          "city" : "Hobucken",
          "state" : "CO"
        },
        "sort" : [		#the document’s sort position (when not sorting by relevance score)
          0
        ]
      },
      {
        "_index" : "bank",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : null,
        "_source" : {
          "account_number" : 1,
          "balance" : 39225,
          "firstname" : "Amber",
          "lastname" : "Duke",
          "age" : 32,
          "gender" : "M",
          "address" : "880 Holmes Lane",
          "employer" : "Pyrami",
          "email" : "amberduke@pyrami.com",
          "city" : "Brogan",
          "state" : "IL"
        },
        "sort" : [
          1
        ]
      }...
    ]
  }
}
```

Each search request is self-contained: Elasticsearch does not maintain any state information across requests. To page through the search hits, specify the `from` and `size` parameters in your request.

For example, the following request gets hits 10 through 19:

```sh
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": { "match_all": {} },
  "sort": [
    { "account_number": "asc" }
  ],
  "from": 10,
  "size": 10
}	
'
	
```

To search for specific terms within a field, you can use a `match` query. For example, the following request searches the `address` field to find customers whose addresses contain `mill` or `lane`:

```sh
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": { "match": { "address": "mill lane" } }
}
'

```

To perform a phrase search rather than matching individual terms, you use `match_phrase` instead of `match`. For example, the following request only matches addresses that contain the phrase `mill lane`:

```sh
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": { "match_phrase": { "address": "mill lane" } }
}
'

```

To construct more complex queries, you can use a `bool` query to combine multiple query criteria. You can designate criteria as required (must match), desirable (should match), or undesirable (must not match).

For example, the following request searches the `bank` index for accounts that belong to customers who are 40 years old, but excludes anyone who lives in Idaho (ID):

```sh
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
      "must": [
        { "match": { "age": "40" } }
      ],
      "must_not": [
        { "match": { "state": "ID" } }
      ]
    }
  }
}
'

```

Each `must`, `should`, and `must_not` element in a Boolean query is referred to as a query clause. How well a document meets the criteria in each `must` or `should` clause contributes to the document’s *relevance score*. The higher the score, the better the document matches your search criteria. By default, Elasticsearch returns documents ranked by these relevance scores.

The criteria in a `must_not` clause is treated as a *filter*. It affects whether or not the document is included in the results, but does not contribute to how documents are scored. You can also explicitly specify arbitrary filters to include or exclude documents based on structured data.

For example, the following request uses a range filter to limit the results to accounts with a balance between \$20,000 and $30,000 (inclusive).

```sh
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
      "must": { "match_all": {} },
      "filter": {
        "range": {
          "balance": {
            "gte": 20000,
            "lte": 30000
          }
        }
      }
    }
  }
}
'
	
```

