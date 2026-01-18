---
title:  'Prefix Query Performance'
slug:  'prefix-query-performance'
date: '2017-02-20'
aliases:
  - /2017/02/prefix-query-performance'
draft: false
---
Last month GitHub released a new feature enabling global search over all
[commit messages](https://github.com/blog/2299-search-commit-messages). One
interesting aspect of this release is the ability to search for a commit by its
SHA1 hash or just the first several characters of the SHA1. This post describes
how we implemented this type of partial matching.

The two methods we have used in the past for this type of search are:

* prefix queries
* and leading-edge ngram analysis

Leading-edge ngram analysis is performed at indexing time as commits are added
to the search index. The commit SHA1 is broken into subsequently longer tokens
anchored to the beginning of the SHA1 string. With this approach each SHA1
generates 40 tokens. The advantage is that queries are very fast at the expense
of an increase in index size. It would be nice to avoid the index bloat, so
let's look at prefix queries instead.

Prefix queries do not require any special analysis at indexing time. We store
the commit SHA1 as a non-analyzed string. Prefix queries work by finding matching
terms in the postings list and collecting all the document IDs associated with
those terms. If there are "many" terms that match the prefix, then this type of
search can be slow. So let's get some numbers!

The commits search index currently has ~6 billion commit documents spread across
32 shards. The postings list is maintained in sorted order, and a binary search
algorithm is used to find terms in the postings list. The binary search has
`O(log n)` performance. Each shard is searched in parallel, and it will take ~20
comparisons to find a term in the worst case scenario. So this is not a show
stopper.

Now lets look at the "many" terms that might possibly match the prefix. We use 7
characters as the lower limit for these short SHA1 strings. With 16 hexadecimal
characters we have `16^7 - 1` unique short SHA1 strings (this is ~270 million
unique strings). If we divide our ~6 billion commit records by this number of
unique short SHA1 strings, we arrive at 22 commits that could match each of
these short SHA1 strings.

So does 22 qualify as "many" prefix match terms? Thankfully it does not. We also
spread these 22 possible matches around the 32 shards in the index. So on
average we will only have one SHA1 string in the postings list that will match
this 7 character short SHA1 prefix.

We'll use commit [`2267b44`](https://github.com/TwP/logging/commit/2267b44c18fa54986c38a1d3ceff84b1785991a9)
as an example for testing out this prefix query. You can
[search for this commit](https://github.com/search?q=2267b44&type=Commits) and
find 5 results. Running this directly against our search cluster we find ...

```sh
$ curl -s -XGET 'localhost:9200/commits/commit/_search' -d '{
  "query": {
    "prefix": {
      "hash": "2267b44"
    }
  }
}' | jq '.'

{
  "took": 10,
  "timed_out": false,
  "_shards": {
    "total": 32,
    "successful": 32,
    "failed": 0
  },
  "hits": {
    "total": 12,
    "max_score": 1,
    "hits": [ ... ]
  }
}
```

So the query took 10ms to complete and found 12 matching commits. This is more
than the 5 results shown on the commits search page because 7 of the 12 results
are from private repositories. It is fun to adjust the length of the short SHA1
string and see how it affects results. Here is small table showing SHA1 length,
number of results, and query time:

| Length | Results | Time |
|----|----|----|
| 4 | 84033 | 21 ms |
| 5 | 7089 | 15 ms |
| 6 | 101 | 9 ms |
| 7 | 12 | 8 ms |
| 8 | 1 | 7 ms |

Prefix queries will definitely be performant enough for our use case. The extra
index size incurred by using leading-edge ngrams is not worth the slight
performance gain the ngrams would provide.

If you would like some further reading on prefix queries and ngrams, there is a
great writeup on partial matching in
[Ealsticsearch: The Definitive Guide](https://www.elastic.co/guide/en/elasticsearch/guide/current/partial-matching.html).
