+++
date = '2025-09-05'
title = 'Tips & tricks for Hugo migration'
draft = true
slug = "tips-for-hugo-migration"
+++

```txt
TODO:
- [ ] Tune the title
- [x] Get the bulk of content in
- [ ] Edit
```

I migrated this site from Jekyll to Hugo over the course of a week during my between-the-job funemployment.

There were couple issues with the previous version:
- It forked a Jekyll bootstrapper ([jekyll-now][jekyll-now]) which felt like an uncesesary layer of abstraction.
- The design felt cluttering, and wanted to make it simple & text focused.
- Comments were hosted on Disqus which I don't use elsewhere.

This post goes over my experience with the migration as well as what I wish I knew before starting the process.

## Understanding Hugo concepts

I found Hugo documentation quite hard to navigate. Often when starting out with a new tool, I'm looking to get an overview of how critical features are tied together. Instead, Hugo documentations is organized into dozen sections where each page have it's own `Introduction` page.

If I was to start over again, I would read Agustin Canalis's excellent [Hugo overview post][hugo-overview] as well as [this Hugo templates guide][hugo-templates] by Régis Philibert first to go beyond the "Quick start" doc.

## Design

Thankfully I found [Hugo bear blog theme][hugo-bear-theme] relatively quickly, so it was a matter of customizing the color palatte to my liking. After rolling the dice with [colors.co][colors] bunch of times, I was satisfied with this [green / orange based pallete][my-pallete]. Once I got a passing grade on [whocanuse.com][whocanuse] and checked its accessiblity, the next focus on specifc CSS issues.

### Code blocks

Besides the minor issues (e.g. inline code highlight, [iOS safari text scaling][ios-safari], etc), the biggest challenge for me was handling the code overflow. When the line went beyond the initial width of the code window, it lacked a background style.

![code_css_issue](/images/2025-09-05-hugo-migration/code_css_issue.png)

Thankfully Veriphor's ["Code block line numbers"][code-block-post] post explained how it's rendered and the solution provided worked for me.

```css
.highlight pre code {
    display: inline-block;
}

.highlight div,
.highlight > pre {
    overflow-x: auto;
}
```

For these code blocks, Hugo use the [Chroma][chroma] syntax highlighter under the hood. When a theme is referenced in `hugo.toml`, it's using the themes from Chroma. I found that not all themes are created equally. In particular, `github-dark` theme incorrectly highlited code so I went with `dracula` instead which was more accurate.

## RSS feed

Previous version relied solely on Jekyll's default RSS feed generator, but I wanted to get a better understanding of the expected format. I first read these two posts to get an overview of how it's hooked up:
- Joseph D Heyburn: ["Who Goes Blogging 6: Three Steps to Improve Hugo's RSS Feeds"][rss-post-1]
- Ben Congdon: ["Tips for Customizing Hugo RSS Feeds"][rss-post-2]

The important pattern to recognize here is that RSS feed can be overwritten with a new `layouts/_default/rss.xml` file. The order of RSS feed processing is in the order of `your own in layouts/_default` > `theme default` > `hugo default`.

Afterwards I pleasantly found out that the RSS spec is short & straightford (["RSS 2.0 Specification"][rss-spec]). Publishing my RSS feed also turned me into a RSS user (with the [Reader app][reader]) which was another unexpected outcome of this migration.

## Image optimization

In my first attempt, I placed all the images under `static/images`. However, I quickly found out that placing them under `assets/images` make it available for Hugo's [image processing capabilities][image-post]. At this point, the easiest way is to use Hugo shortcode to refer to them in the Markdown file. However, I preferred using the markdown native `![ ]( )` format.

One way to get around this is to use [Render Hooks][render-hooks]. For example, when Hugo sees the following markdown:
```txt
# Hello World

![earth][/images/earth.png]
![sun][/images/sun.png]]
```

It recognizes that the file requires rendering `Heading` and two `Image` components. For each of these types, Hugo checks `layouts/_default/_markup` directory for user created render hooks. In my case, I created the following in `layouts/_default_markup/render-image.html` (note that the file name needs to match exactly as expected in the doc):

```html
{{- /* Use the render hook to make asset images available in posts */ -}}
{{- $image := resources.Get .Destination -}}
{{- if $image -}}
  {{- if strings.HasSuffix $image ".gif" -}}
    <img src="{{ $image.RelPermalink }}" alt="{{ .Text }}"{{- with .Title }} title="{{ . }}"{{ end }} loading="lazy" />
  {{- else -}}
    {{- $optimized := $image.Process "webp q75 lanczos" -}}
    <img src="{{ $optimized.RelPermalink }}" alt="{{ .Text }}"{{- with .Title }} title="{{ . }}"{{ end }} loading="lazy" decoding="async" />
  {{- end -}}
{{- end -}}
```

The top level dot `.` in this case refers to the [Render Hook Image context][image-context] which includes the `Destination` method for the image path in `assets/`. `resources.Get` returns the [image resource][resource-doc] context. It then calls the [`Process` method][process-method] to request the optimized version (`"webp q75 lanczos"`). This render hook is called per `Image` component discovered during the markdown rendering process. In the example Markdown text, it would be called twice.

Michael Welford's ["Responsive image management with Hugo"][image-blog-post] goes much beyond my basic implementation if you want to expand on it.

Using the simple image processing allowed one of my image-heavy posts to reduce total image size from 673 KB to 176 KB total which is a 73% reduction!

## Comment system

Darek Kay wrote a [great post][hugo-comment] on the current static site comment ecosystem. I initially wanted to implement Bluesky commenting system (like in Kaushik Gopal's [post][bluesky-post]), but realized that my reader would be technical folks so using [Giscus][giscus] based on GitHub is lower barrier of entry. The whole setup took maybe 10 minutes.

I recommend reading Bryce Wray's ["Tips for using giscus"][giscus-tips] post. The point about creating separate repo just for comment discussions was something I would have overlooked overwise.

## Deployment

This site is hosted on GitHub Pages & deployed with GitHub Actions. I copied & pasted the YAML from the [deployment documentation][deploy-doc] and it surprisingly worked right away. The only hiccups was the extra commits to the theme git submodule I made which CI couldn't pull from, so I manually moved the theme under `themes` directory instead.

The domain's DNS nameserver is set to Cloudflare for basic DDoS protection, TLS, bot filtering, and CDN caching. I moved from Google Analytics to more privacy-focused [Simple Analytics][simple-analytics] to avoid setting up the cookie banner.

## Summary

Although it was certainly a bumpy ride, I'm still content with the decision to go with Hugo. It had all the features I needed (e.g. Image Processing) and CLI was as fast as people raved about. I also unexpectedly found that Hugo users seem to be eager to share what they learned on their blogs! This post is my way of paying it forward.

Let me know if any of the tips were useful in the comments!

[jekyll-now]: https://github.com/barryclark/jekyll-now?tab=readme-ov-file
[colors]: https://colors.co
[hugo-overview]: https://acanalis.github.io/post/concepts-of-hugo/
[hugo-templates]:  https://www.regisphilibert.com/blog/2018/02/hugo-the-scope-the-context-and-the-dot/
[hugo-bear-theme]: https://github.com/janraasch/hugo-bearblog
[my-pallete]: https://coolors.co/db9d47-ff784f-ffe19c-ecf4e4-1b2920
[whocanuse]: https://www.whocanuse.com
[ios-safari]: https://stackoverflow.com/questions/3226001/some-font-sizes-rendered-larger-on-safari-iphone
[code-block-post]: https://www.veriphor.com/articles/code-block-line-numbers/
[rss-spec]: https://cyber.harvard.edu/rss/rss.html
[rss-post-1]: https://jdheyburn.co.uk/blog/who-goes-blogging-6-three-steps-to-improve-hugos-rss-feeds/
[rss-post-2]: https://benjamincongdon.me/blog/2020/01/14/Tips-for-Customizing-Hugo-RSS-Feeds/
[reader]: https://readwise.io/read
[image-post]: https://hugo-mini-course.netlify.app/sections/optimizing/images/
[chroma]: https://github.com/alecthomas/chroma
[render-hooks]: https://gohugo.io/render-hooks/introduction/
[image-context]: https://gohugo.io/render-hooks/images/#context
[resource-doc]: https://gohugo.io/methods/resource/
[process-method]: https://gohugo.io/methods/resource/process/
[image-blog-post]: https://its.mw/posts/hugo-responsive-images/
[hugo-comment]: https://darekkay.com/blog/static-site-comments/
[bluesky-post]: https://kau.sh/blog/bluesky-comments-for-hugo/
[giscus]: https://giscus.app/
[giscus-tips]: https://www.brycewray.com/posts/2022/05/tips-using-giscus/
[deploy-doc]: https://gohugo.io/host-and-deploy/host-on-github-pages/
[simple-analytics]: https://www.simpleanalytics.com/
