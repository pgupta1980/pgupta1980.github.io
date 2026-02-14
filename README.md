# Prasenjit Gupta's Blog

This is the source code for my personal blog, built with Jekyll and hosted on GitHub Pages.

## Local Development

```bash
# Install dependencies
bundle install

# Run local server
bundle exec jekyll serve

# Visit http://localhost:4000
```

## Writing Posts

Create new posts in the `_posts/` directory with the format:
```
YYYY-MM-DD-title-of-post.md
```

Include front matter:
```yaml
---
layout: post
title: "Your Post Title"
date: YYYY-MM-DD HH:MM:SS -0000
categories: category1 category2
---
```

## Deployment

Push to the `main` branch and GitHub Pages will automatically build and deploy your site.

## Live Site

Visit: [https://pgupta1980.github.io](https://pgupta1980.github.io)
