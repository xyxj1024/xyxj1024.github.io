# Xingjian Xuanyuan's Home Page

Please feel free to [contact me](mailto:xyxj16@tsinghua.org.cn) if there are any deprecated links or other issues with the contents of the website.

Any incident with GitHub Actions, Codespaces, and Pages is reported on [https://www.githubstatus.com/](https://www.githubstatus.com/).

## To-Dos

- Hardware Security. Foreshadow attack.
- Hardware Security. [Hasan Hassan](https://safari.ethz.ch/congratulations-to-hasan-hassan-on-his-edaa-outstanding-dissertation-award-2023/) and [Onur Mutlu](https://people.inf.ethz.ch/omutlu/)'s collaborative works on DRAM.
- Cybersecurity. Small subgroup confinement attack on Diffie-Hellman.
- Programming Languages. Golang's [GC](https://github.com/golang/go/blob/master/src/runtime/mgc.go) mechanism.
- Page Design. `Blog Posts` page categories reordering: Is there a simple way to sort `site.categories` by post count using Liquid syntax?
- Page Design. Discussion of the integration of Chinese fonts to GitHub Pages by Long Qian in [this post](http://longqian.me/2017/02/12/jekyll-support-chinese/). However, since the proposed approach requires too much code rewrite, I decide to use system fonts for now.

## Acknowledgement

Many thanks to:
- Christian Fei's [JavaScript library](https://github.com/christian-fei/Simple-Jekyll-Search) that enables the search box functionality.
- Ari Stathopoulos's idea of a [commenting system](https://aristath.github.io/blog/static-site-comments-using-github-issues-api) leveraging the GitHub Issues API (available to GitHub users).

## Development

Created on top of [Hyde](https://github.com/poole/hyde), a brazen two-column [Jekyll](http://jekyllrb.com/) theme that pairs a prominent sidebar with uncomplicated content. (You might find quite a few people using this template :sweat_smile:)

For HTML codes of standard [Chinese punctuation marks](https://en.wikipedia.org/wiki/Chinese_punctuation), [this W3C document](https://lists.w3.org/Archives/Public/public-clreq-admin/2015JulSep/att-0000/index-dual.html) provides a good summarization.

For [this Liquid syntax error](https://github.com/kubevirt/kubevirt.github.io/issues/598), make sure to use the <code>{% raw %}</code> and <code>{% endraw %}</code> escapes to force Jekyll to ignore the code block, as documented [here](https://github.com/kubevirt/kubevirt.github.io/pull/603).

Sometimes need a hard reload of the web page using `ctrl/cmd + shift + r`.