---
layout: post
title:  "Versioning ElasticSearch Indices"
---

Here is the approach we use at GitHub to update the search indices on our
developer's machines and in our test environment. We want this process to be
automatic and painless. Developers should not need to take special actions
when search indices are updated - it needs to just work on their machine.

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

Our index mappings are literal Ruby Hashes that live in source code files.
When the mapping is updated or changed, the corresponding index needs to be
recreated and populated. Here is where the version SHA1 pays off.

The changes in the mapping results in a new version SHA1. This will be
different than the SHA1 stored in the index settings in ElasticSearch.

```bash
curl -XGET 'http://localhost:9200/demo/_settings'
```

When these two version SHA1s disagree, our bootstrap script knows to recreate
and repopulate the index. Because of the version SHA1s, only those indices
that have been modified are recreated. And all of this happens without any
extra work required from our developers.

Here's to reducting friction!

### Why Use a SHA1?

Wouldn't a version number be simpler? Well yes ... and no. We could put an
explicit version number in the settings for each index. The settings, like our
mappings, are also literal Ruby Hashes that live in source code files. You
could look at the source code and easily see that "this is version 42 of the
index."

The problem is remembering to change that version number each time you make a
change to the index. Forcing a human to remember anything is error prone - it
increases friction.

With the SHA1 the whole versioning process is automated. There is nothing to
forget, and we can focus on improving search - not managing search.

Thanks for reading.
