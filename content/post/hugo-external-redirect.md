---
title: "Hugo: External Redirects"
date: 2026-09-11
draft: false
slug: "external-redirects-hugo"
cover:
  image: "https://canada1.discourse-cdn.com/flex036/uploads/gohugo/original/2X/b/b91c8ab3a3c1c8679127cf049b46fa919e9e0e5c.png"
categories: ["Tech"]
tags: ["en", "hugo", "web"]
---

## The Why

The majority of technical and article-worthy work I do happens at my daily job. I don't think this would sound surprising to anyone. Luckily, my current employer is also quite welcoming when it comes to writing technical articles. Obviously, these articles then go to the corporate blog, which is fair.

However, this made me think, how do I link those articles to my personal blog properly? Previously, I would just link those articles somewhere on the [About](https://grem1.in/about/) page labeled as "external articles" and whatnot. Obviously, no one would find them there. Apparently, I am not the only one who stumbled upon this question. I found a couple of resources online on how to enable external redirects in Hugo, such as "[External Redirects with Hugo](https://nickherrig.com/posts/hugo-external-redirects/)" by Nick Herrig, and [this request](https://discourse.gohugo.io/t/redirect-post-to-external-url/44138) on the Hugo support forum.

Unfortunately, neither of those ways worked for me. Fortunately, the solution was very similar and rather simple. So, since I couldn't find the exact solution online, I decided to put it into writing myself.

## The How

Just like in the Mr. Herrig's case, I used [Hugo's layouts](https://gohugo.io/templates/lookup-order/#target-a-template) to create redirects. Yet, I haven't modified any of the internal files or any of the theme files. Thus, this approach should be theme-agnostic, at least in theory. It works for me with a fresh [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme and Hugo `v0.165.0`, which is quite recent at the time of writing of this article.

If you don't have a directory with layouts, you need to create a new `layouts/` directory with a named layout in your Hugo root, for example:

```bash
mkdir -p layouts/_default
```

Then create a new HTML file `redirect.html` with the following content:

```html
{{- $url := .Params.redirectURL -}}
<!DOCTYPE html>
<html lang="{{ site.Language.LanguageCode | default "en" }}">

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{{ .Title }}</title>
  {{- with $url }}
  <meta http-equiv="refresh" content="0; url={{ . }}">
  <link rel="canonical" href="{{ . }}">
  <meta name="robots" content="noindex, follow">
</head>

<body>
  {{- with $url }}
  <p>Redirecting to <a href="{{ . }}">{{ . }}</a>&hellip;</p>
  <script>window.location.replace({{ . }});</script>
  {{- else }}
  <p>Missing <code>redirectURL</code> in front matter.</p>
  {{- end }}
</body>

</html>
```

`Meta` tag does all the redirect magic here. The important things here are the file name and that `.Params.redirectURL` arguments. The file name is important, because this is how you are going to mark an article for an external redirect. `.Params` is [the front matter map of custom parameters](https://gohugo.io/methods/page/params/#article) that exists for any Hugo page. Technically, you can call the variable whatever you want as long as it doesn't overlap with any default parameter. `redirectURL` is verbose enough yet also short enough, in my opinion.

## The What

Now, with all the machinery in place, you can create your external redirect article! Here's the content example for the one of mine:

```yaml
---
redirectURL: "https://medium.com/preply-engineering/load-testing-apollo-router-2fe3c93e43c7"
layout: redirect
title: "Load Testing Apollo Router"
date: "2026-08-27"
cover:
  image: "https://miro.medium.com/v2/resize:fit:4800/format:webp/1*XGzFMRY_H-DVcp7mpmXv5w.png"
---

How we ran a load test for Apollo Router to figure out its limitations and performance gains
compared to the legacy systems. 
```

As you can see, it has all your normal metadata, such as `title` and `date`. Of course, you can add more metadata if you need. The main things are `layout: redirect`, which calls our `redirect.html` layout by name, and `redirectURL`, which tells the browser where to redirect a user.

As you can see, I also have some text in there. You don't need to have any, but the theme I use - PaperMod - adds some preview text on an article tile that is taken from the first paragraph. So, I added some to hint people on what to expect from the article.

Now, you should be all set!
