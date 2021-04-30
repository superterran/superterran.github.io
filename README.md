# superterran.github.io

A static website using the hugo static site generator, with the future-imperfect-slim theme, hosting on github.io. 

## Local Usage

```/bin/bash
hugo server --watch -D
```

## Customizations

Ended up deviating from the vanilla Hugo setup in a few places...

### Configuration 
* Removes requirement for front-matter for blogposts
* Hides social buttons on posts
* Tweaks permalink structure to only rely on filename

### Layout Changes

* Infers `date` field from filesystem unless provided
* Pulls `title` from filename if field isn't populated in front-matter

## Links

* [Hugo Reference Guide](https://gohugo.io/getting-started/quick-start/)
* [formspree](https://formspree.io/forms/xyylkwoz/submissions)
