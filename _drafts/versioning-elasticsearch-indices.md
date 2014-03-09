---
layout: post
title:  "Versioning ElasticSearch Indices"
---

Here is the approach we use at GitHub to update the search indices on our
developer's machines and in our test environment. We want this process to be
automatic and painless. Developers should not need to take special actions
when search indices are updated - it needs to just work.

### Generating Versions

We start by assigning a version to each index. This is a SHA1 digest of the
settings and mappings for the index. A simple value `v` is also mixed in to
the digest. This value enables us to force an index to be recreated without
actually modifying the settings or mappings. This is a useful feature in our
continuous integration environment.

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

### Storing Versions

Here is the clever bit. We can add **any** field to the index settings, and
ElasticSearch will store the field. Let's store our version SHA1 in the index
settings when it is created.

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

Our index mappings are literal Ruby Hashes that live in source code files.
When the mapping is updated or changed, the corresponding index needs to be
recreated and populated. Here is where the version SHA1 pays off.

The changes in the mapping results in a new version SHA1. This will be
different than the SHA1 stored in the index settings in ElasticSearch.

```bash
curl -XGET 'http://localhost:9200/demo/_settings'
```

When these two version SHA1s disagree, our bootstrap script knows to recreate
and repopulate the index. And only those indices that have been modified are
recreated. And all of this happens without any extra action from our
developers.

Here's to reducting friction! Thanks for reading.

