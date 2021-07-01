---
title: "My Markup test"
date: 2021-06-28T14:45:02+08:00
draft: false
tags: ["python"]
categories: ["Network"]

---



## 1 Content1

##Test


#ok

## 2 Content2

```html
<section id="main">
    <div>
        <h1 id="title">{{ .Title }}</h1>
        {{ range .Pages }}
            {{ .Render "summary"}}
        {{ end }}
    </div>
</section>
```

{{< highlight html "linenos=table,linenostart=199" >}}
<section id="main">
    <div>
        <h1 id="title">{{ .Title }}</h1>
        <h2>ok<h2>
    </div>
</section>
{{< / highlight >}}

This is a **right-aligned** paragraph.


```python
rows = int(input("Enter number of rows: "))

k = 0

for i in range(1, rows+1):
    for space in range(1, (rows-i)+1):
        print(end="  ")
   
    while k!=(2*i-1):
        print("* ", end="")
        k += 1
   
    k = 0
    print()
```

## 3 Content


> **Fusion Drive** combines a hard drive with a flash storage (solid-state drive) and presents it as a single logical volume with the space of both drives combined.

Gone camping! :tent: Be back soon.
That is so funny! :joy:
