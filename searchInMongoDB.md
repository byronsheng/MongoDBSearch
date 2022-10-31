# Build Complex Index and Atlas Search Synax in MongoDB

## EmbeddedDocument and Autocomplete, fuzzy search for full text search in MongoDB

---

### 1. Index and search synax in MongDB

You gonne need to have some understanding about index and search synax in MongDB.
 * Index is what and where you want to search, you need to define it, to let the database know, how to deal and order with your data.
 * search synax is the language or Query language for searching in MongDB

---

e.g: Data schema can be

```
class YourObject {
	List<YourSubObject> YourSubObject
	string Property1
	string Property1
	string Property1
}

class YourSubObject{
	List<YourSub-SubObject> YourSub-SubObject
	string SubProperty1
	string SubProperty2
}

class YourSub-SubObject{
	string Sub-SubProperty1
	string Sub-SubProperty2
}
```

if you want to search something in your Sub-SubProperty1 with autocomplete and full text search-
You need to define all the path deep to your Sub-SubProperty and the operator in synax is not "autocompele" but is "embeddedDocument".

so here is  a beispiel for the index, with the index you can $search the sub-SubProperty1 as **autocomplete**

```
{
  "mappings":
  {
    "dynamic" : false,
    "fields":
    {
      "yoursubObject":
      {
        "type": "embeddedDocuments",
        "fields":
        {
          "YourSub-SubObject":
          {
            "type":"embeddedDocuments",
            "fields":
            {
              "sub-SubProperty1":
              [
                {
                  "type" : "string"
                },
                {
                  "type": "autocompele",
                  "foldDiacritics": true,
                  "minGrams": 3,
                  "tokenization": "nGram"
                }
              ]
            }
          }
        }
      }
    }
  }
}
```

by using syntax you may get the result you want, here is the beispiel to use autocompele in mongodb:
start with $search

```
{
  "embeddedDocument":
  {
    "path": "yourSubObject",
    "operator" : "embeddedDocument" : {
      "path" : "yourSubObject.yourSub-SubObject",
      "operator": "autocomplete" : {
        "path": "yourSubObject.yoursub-SubObject.sub-SubProperty1",
        "query": "testQueryString"
      }
    }
  }
}

```
you can find that, you need to always define the operator as **embeddedDocument** to search deep in the arrays of your Objects, and for the path of sub-SubProperty1, your operator is finally **autocomplete**.

***How to Run Atlas Search Queries Against Objects in Arrays:***
https://www.mongodb.com/docs/atlas/atlas-search/tutorial/embedded-documents-tutorial/

***Create an Indenx:***
https://www.mongodb.com/docs/atlas/atlas-search/create-index/

---

### 2. advanced Knowlage of Index and search syntax

now we can go further to the Index and search syntax. there is a lots of "parameters" in Index and search syntax, to make your search better, you need to make edit and test, till you get the result, which you want.

#### Index

lets start with the indexes, also with the example above. Here is the parameters for an index:

| Syntax      | Description |
| ----------- | ----------- |
| Index Analyzer  | Creates searchable terms from data to be indexed.|
| Search Analyzer | Parse $search queries into searchable terms. |
| Dynamic Mapping | Automatically index common data types in a collection |
| Store Full Document | Make all data available for lookup on Atlas Search side |

<mark>analyzer:

Uses the default analyzer for all Atlas Search indexes and queries.

>Simple: Divides text into searchable terms wherever it finds a non-letter character.

>Whitespace: Divides text into searchable terms wherever it finds a whitespace character.

>Language: Provides a set of language-specific text analyzers.

>Keyword: Indexes text fields as single terms.

>standard: It divides text into terms based on word boundaries, which makes it language-neutral for most use cases

<mark>searchAnalyzer:

Specify the analyzer to use when parsing data for the Atlas Search queries. By default, Atlas Search uses the standard analyzer ("lucene.standard").

<mark>Process Data with Analyzers:<mark>

* Analyzers: Standard, Simple, Whitespace, Language, Keyboard
* Use different Analyzers with different languages

```
{
  "analyzer": "lucene.whitespace",
  "searchAnalyzer": "lucene.german",
  "mappings": {
    "dynamic": true,
    "fields": {
      "addresses": [
        {
          "type": "document",
          "dynamic": false
        },
        {
          "type": "string"
        }
      ]
    }
  },
  "storedSource": true
}
```
<mark>Multi Analyzer

* you can define multi analyzer in your index, but you need to also use it by Syntax, see: https://www.mongodb.com/docs/atlas/atlas-search/analyzers/multi/


then you can config the index:

```
{
    "mappings": {
      "dynamic": false,
      "fields": {
        "title": {
          "type": "string",
          "analyzer": "lucene.standard",
          "multi": {
            "keywordAnalyzer": {
              "type": "string",
              "analyzer": "lucene.keyword"
            }
          }
        }
      }
    }
  }
```

#### Syntax

Atlas Search queries take the form of an aggregation pipeline stage, we start with $search.

Definition:
The $search stage performs a full-text search on the specified field or fields which must be covered by an Atlas Search index.

```
{
  $search: {
    "index": "<index-name>",
    "<operator-name>"|"<collector-name>": {
      <operator-specification>|<collector-specification>
    },
    "highlight": {
      <highlight-options>
    },
    "count": {
      <count-options>
    },
    "returnStoredSource": true | false
  }
}
```
Fields:

| Field      | Description|
| ----------- | ----------- |
| collector-name | Name of the collector to use with the query. You can provide a document that contains the collector-specific options as the value for this field. Either this or "operator-name" is required. |
| index | Name of the Atlas Search index to use. If omitted, defaults to default. |
| operator-name | Name of the operator to search with. You can provide a document that contains the operator-specific options as the value for this field. Either this or "collector-name" is required. |
| returnStoredSource | Flag that specifies whether to perform a full document lookup on the backend database or return only stored source fields directly from Atlas Search. If omitted, defaults to false.  |

we have knew, how to create a syntax in the example above, but there are more configrations, that can make your search result better. We just take an ***Operator*** of "autocomplete". In the "type" of "autocomplete", you may config the fields of ***tokenization***, ***minGrams***, ***maxGrams***, ***foldDiacritics***.

```
{
  "type": "autocomplete",
  "tokenization": "edgeGram",
  "minGrams": 3,
  "maxGrams": 7,
  "foldDiacritics": false
}
```

***!!! important: Limitaions of Autocomplete:***

The autocomplete operator query results that are exact matches receive a lower score than results that aren't exact matches. Atlas Search can't determine if a query string is an exact match for an indexed text if you specify just the autocomplete-indexed token substrings. So we need to combine the "text" search and the "autocomplete", in order to get higher Score when the string exact matches.

To combine "text" and "autocomplete" in search, you need to in the index add type as "string" in the field and in the syntax use "compound" operator.

#### Fuzzy search

the definition of fuzzy search: Enable fuzzy search. Find strings which are similar to the search term or terms. You can't use fuzzy with synonyms.
https://www.mongodb.com/docs/atlas/atlas-search/text/#fuzzy-examples

fuzzy search is an option for "text" search or "autocompele" search, here is an example in text search:

```
{
  $search: {
    "index": <index name>, // optional, defaults to "default"
    "text": {
      "query": "<search-string>",
      "path": "<field-to-search>",
      "fuzzy": <options>,
      "score": <options>,
    }
  }
}

```

and in "fuzzy" you can config:

| Field      | Description |
| ----------- | ----------- |
| fuzzy.maxEdits  | Maximum number of single-character edits required to match the specified search term. Value can be 1 or 2. The default value is 2. Uses Damerau-Levenshtein distance. |
| fuzzy.prefixLength | Number of characters at the beginning of each term in the result that must exactly match. The default value is 0. |
| fuzzy.maxExpansions | The maximum number of variations to generate and search for. This limit applies on a per-token basis. The default value is 50. |

the example with fuzzy search:

```
{
    $search: {
      "text": {
        "path": "title",
        "query": "naw yark",
        "fuzzy": {
          "maxEdits": 1,
          "prefixLength": 2
        }
      }
    }
}
```

---

### how to use Pipeline and BsonDocument in C# ??

you will get a very long Codes, when your fields and conditioins are complex.
how to use it without magic string and maybe easier is still a problem.
Use function in app Service of MongoDB may be a solution, but the performance is not as good as the Pipeline Search.

---

#### more tutorials:
$search vs $regex vs $text: The Best Way to Get There: https://www.youtube.com/watch?v=-sRcpGpd-0s

$search will have more performance and for the user experience, is the $search the best, cause of feature of "autocomplete" and fuzzy "search"

autocomplete documents:
https://www.mongodb.com/docs/atlas/atlas-search/autocomplete/

