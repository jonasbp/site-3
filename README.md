# Convox Docs <img src="https://convox.com/assets/images/logo.png" width="40%">

[![Build Status](https://travis-ci.org/convox/site.svg?branch=master)](https://travis-ci.org/convox/site)

This repo contains the source files for [convox.com](https://convox.com/).

Documentation is written in Markdown (with options similar to [Github's Flavor](https://help.github.com/articles/github-flavored-markdown/)).

[Jekyll](https://http://jekyllrb.com//) is used to generate static HTML files.

Please let us know about any issues, either via [issues](/issues) or by emailing [support@convox.com](mailto:support@convox.com)

Pull requests are also welcome!

## Usage

Convox and Docker are recommended.

```shell
$ git clone https://github.com/convox/site.git && cd site
$ echo DEVELOPMENT=true >> .env
$ convox start
```

### Running directly on your host

Ruby and bundler are required.

```shell
$ bundle install
$ jekyll serve
==    Server address: http://127.0.0.1:4000/
```

Please send updates to documentation as [Pull Requests](/pulls).

### Frontmatter

Each page in the documentation has a header, called frontmatter. This controls where it's displayed and its title.

e.g.

```
---
title: "Custom Domains"
sort: 40
---
```

# Publishing

When a pull request is merged into the master branch of this repo, the change is automatically deployed to [staging](http://site-staging.convox.com/).

Then, once one of the maintainers of this repo promotes the release on the Convox production rack, it will be visible on [convox.com](https://convox.com).
