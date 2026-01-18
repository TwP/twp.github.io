---
date: '2026-01-02T11:47:40-07:00'
draft: true
title: 'My First Post'
slug: 'my-first-post'
menu: "archive"
Description: ""
Tags: ["Development", "golang"]
Categories: ["Development", "GoLang"]
---
This is some **bold** text, and this is some *italicized* text.

```go {linenos=inline hl_lines=[3,"6-8"] style=manni}
package main

import "fmt"

func main() {
    for i := 0; i < 3; i++ {
        fmt.Println("Value of i:", i)
    }
}
```

---

```ruby {linenos=false}
def version( settings, mappings, v = 0 )
  json = MultiJson.dump \
    v: v,
    s: settings,
    m: mappings

  sha = Digest::SHA1.hexdigest json
  sha.freeze
end
```


Try [Kagi](https://kagi.com) as your search engine.