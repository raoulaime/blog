# Parked blog

Posts, diagrams and the home page block removed on 2026-11-08, to bring back later.

To restore:
1. `mv _parked/blog content/blog` and `mv _parked/img static/img`
2. Add the Blog entry back to the menu in config.toml (weight 2)
3. Put the "Latest writing" section back in layouts/index.html, the RSS link in layouts/partials/footer.html and the alternate link in layouts/_default/baseof.html

Home page block:

```html
<section class="wrap block">
  <h2>Latest writing</h2>
  {{ $posts := (where .Site.RegularPages "Type" "blog").ByDate.Reverse }}
  {{ with index $posts 0 }}
  <a class="featured" href="{{ .RelPermalink }}">
    {{ with .Params.image }}<img src="{{ . }}" alt="" loading="lazy">{{ end }}
    <div><p class="org">Featured</p><h3>{{ .Title }}</h3><p>{{ .Description }}</p><span class="meta">{{ .Date.Format "Jan 2, 2006" }} · {{ .ReadingTime }} min read</span></div>
  </a>
  {{ end }}
  {{ partial "postlist.html" (first 3 (after 1 $posts)) }}
  <p><a href="/blog/">All posts →</a></p>
</section>

```
