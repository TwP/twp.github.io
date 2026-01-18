---
title:  'Versioning ElasticSearch Indices'
slug:  'versioning-elasticsearch-indices'
date: '2014-03-08'
aliases:
  - /2014/03/versioning-elasticsearch-indices
draft: false
---
Working with search is a process. You create an index and build queries and
realize they need to change. So how do you manage that process of change
without driving everyone you work with insane?

Here is the approach we use at GitHub to update search indices on our
developer's machines (and also in our test environment). We want this process
to be automatic and painless. Developers should not need to take special
action when search indices are updated - it needs to "just work".

### Generating Versions

We start by assigning a version to each index. This is a SHA1 digest of the
settings and mappings for the index. A simple value `v` is also mixed in to
the digest. This value enables us to force an index to be recreated without
actually modifying the settings or mappings.

```ruby
def version( settings, mappings, v = 0 )
  json = MultiJson.dump \
    v: v,
    s: settings,
    m: mappings

  sha = Digest::SHA1.hexdigest json
  sha.freeze
end
```

The `settings` and the `mappings` are both a Hash. Because a Hash in Ruby is
ordered, this version method will return the same SHA1 for the same inputs.

We store the settings and mappings as Ruby Hashes in source code files. We
treat these as the **true** values. If an index in ElasticSearch has different
settings or mappings, then it is considered out of date.

### Storing Versions

Here is the clever bit. We can add **any** field to the index settings, and
ElasticSearch will store the field. Let's store our version SHA1 in the index
settings when the index is created.

```bash
curl -XPUT 'http://localhost:9200/demo/' -d '{
  "settings": {
    "index._meta.version": "SHA1",
    "index": {
      ...
    }
  },
  "mappings": {
    ...
  }
}'
```

After we generate the version SHA1 it is added to the settings Hash under the
`index._meta.version` key. We decided on the `_meta` namespace to contain
metadata about the index. Other fields can be added to this namespace if the
need arises.

### Checking Versions

When a mapping is updated or changed in our source code, the corresponding
ElasticSearch index needs to be recreated and populated. Here is where the
version SHA1 pays off.

The changes in the mapping result in a new version SHA1. This will be
different than the SHA1 stored in ElasticSearch.

```bash
curl -XGET 'http://localhost:9200/demo/_settings'
```

When these two version SHA1s disagree, our bootstrap script knows to recreate
and repopulate the index. And all of this happens without any extra work
required from our developers.

Here's to reducting friction!

### Why Use a SHA1?

Wouldn't a version number be simpler? Well yes ... and no. We could put an
explicit version number in the settings Hash for each index. You could then
look at the source code and easily see that "this is version 42 of the index."

The problem is remembering to change that version number each time you make a
change to the settings or mappings. Forcing a human to remember anything is
error prone - it increases friction.

With the SHA1 the whole versioning process is automated. There is nothing to
forget, and we can focus on improving search - not managing search.

Thanks for reading.
