---
title: "Implementing admonitions in Hugo without theme support"
date: 2023-05-15T20:34:23+02:00
tags:
- Hugo
categories:
- Webdev
---
# The issue
I'm a huge fan of admonitions (like this one below), but couldn't find a proper way to implement them on top of a hugo theme (e.g. the one I'm using now) without touching it.  
{{< admonition title="Admonition" bg-color="#00ee88">}}
Hi! I'm an admonition in Hugo in a theme that does not implement them (nor allows the import of CSS)!
{{< /admonition >}}

# Shortcodes!

My approach is based on [Hugo shortcodes](https://gohugo.io/content-management/shortcodes/).  
Unfortunately, searching for info the only blog page I found was broken (and there was no working archived version), so here is what I did:

I created a new shortcode (`layouts/shortcodes/admonition.html`):
```go-html-template
{{ if not (.Page.Scratch.Get "hasAdmonitions")}}
{{ .Page.Scratch.Set "hasAdmonitions" "True" }}
<style>
    {{ $defaultBgColor := "#00ee88"}}
    .admonition{
        border-radius: 5px;
        padding: 0px;
        border-left: 5px solid {{$defaultBgColor}};
        box-shadow: 0 0 .5rem .2rem #00000025;
    }
    .admonition-title-container{
        background-color: {{$defaultBgColor}};
    }
    .admonition-title {
        font-weight: bolder;
        font-size: large;
        backdrop-filter: grayscale(70%) brightness(140%);
        padding: 5px 0 5px 30px;
    }
    @media (prefers-color-scheme: dark) {
        .admonition-title {
            backdrop-filter: grayscale(40%) brightness(40%);
        }
    }
    .admonition-content{
        padding: 10px 0 10px 15px
    }
</style>
{{ end }}

<div class="admonition" {{if .Get "bg-color"}}style="border-left: 5px solid {{ .Get "bg-color" }};"{{end}}>
    <div class="admonition-title-container" {{if .Get "bg-color"}}style="background-color: {{ .Get "bg-color" }};"{{end}}>
        <div class="admonition-title">
            {{ .Get "title" | default "Tip" }}
        </div>
    </div>
        <div class="admonition-content">{{ printf "%s" .Inner | markdownify }}</div>
</div>
```

Of course this is **totally** not inspired by [Material for mkdocs](https://squidfunk.github.io/mkdocs-material/)

## How to use
The green admonitions above is created with this snippet:

```html
{{</*admonition title="Admonition" bg-color="#00ee88"*/>}}
Hi! I'm an admonition in Hugo in a theme that does not implement them (nor custom CSS)!
{{</*/admonition*/>}}
```

and the bg-color is totally optional.  

If you can read the CSS code you can see the "trickery" I did to have a lighter background.  
Not really proud of that to be honest, but it kinda works, so that's good enough. Just remember to use
<i style="background-image:linear-gradient(90deg,rgba(255,0,0,1) 0%,rgba(255,154,0,1) 10%,rgba(208,222,33,1) 20%,rgba(79,220,74,1) 30%,rgba(63,218,216,1) 40%,rgba(47,201,226,1) 50%,rgba(28,127,238,1) 60%,rgba(95,21,242,1) 70%,rgba(186,12,248,1) 80%,rgba(251,7,217,1) 90%,rgba(255,0,0,1) 100%);color:#fff;font-weight:bolder;padding:2px;letter-spacing:3px;text-shadow: #000 0px 0px 2px;">&nbsp;BRIGHT&nbsp;</i> colors, or the filter will hide the backgroin color in the title section

## How it works
Ignoring the CSS for the lighter code (which is just a [backdrop-filter](https://developer.mozilla.org/en-US/docs/Web/CSS/backdrop-filter)), the way this works is pretty easy: we just format some text in some HTML.  
(What I believe) is the most interesting part here is how the CSS is being managed  

Keep in mind that:

1. The template I use does not implements admonitions
2. The template I use does not allow for using user-provided CSS stylesheets
3. I did not want to tamper with the theme in any way

Since I'm not tampering with the theme, I won't put the CSS in the `head` section of the page. This means I need to put the stylesheet in the HTML page.  
I had two possible routes:

- Use the `` `<CSS CODE>`| resources.FromString "css/admonitions.css" | resources.ToCSS}}`` snippet to create a dummy CSS file to reference
- Paste the stylesheet directly in the HTML

Since the stylesheet is so tiny, I just decided to drop it in the HTML body.  

This now poses a new problem: I don't even want to bloat the HTML page with redundant CSS for each admonition: I just want to define the CSS once.  
This is the reason I used the [page scratch (`.Page.Scratch.Get`)](https://gohugo.io/functions/scratch/), which is a "store" where I can define and use page-scoped variables.  

I proceeded to create a check for a variable that I instantiate only once I've dropped the CSS in the page in the first place.

There you have it, enjoy!