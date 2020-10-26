+++
title = "Leverage Huffduffer from mobile safari (again)"
date = 2020-10-25T07:52:55+02:00
tags = ["js", "scriptable", "ios", "huffduff", "podcast", "rss"]
+++

Swiping through my timeline it sometimes happens that I stumble upon a podcast episode that sounds especially interesting. The issue is, I don't necessarily wanna subscribe to the full thing but only 'd like to listen to that specific episode. 

While this issue is a solved problem in general (_thanks to an awesome service called [Huffduffer][huffduffer]_) it lately became an issue again on my mobile phone (_where I tend to consume my timeline_) since my beloved [huffduffer-workflow][jancbeck] stopped working and the all-new [shortcuts][shortcuts]-app ... well, it felt clunky already [and over time got even worse][issue].
<!--more-->
But some days ago I stumbled upon the [scriptable][scriptable]-app and I saw a chance to bring back my `huffduffit`-share-sheet action. Although I wasn't patient enough to find a good getting-starting tutorial I was able to craft a solution that did the job pretty quickly.

You can find the [resulting script here][huffduffit] - just import it to [scriptable][scriptable] and you're good to go:

{{< figure width="100%" src="/posts/huffduffit/huffduffit.png" alt="Usage of the HuffDuff.it action">}}

If you're frustrated with [shortcuts][shortcuts] - give [scriptable][scriptable] a try!

[jancbeck]:https://jancbeck.com/articles/huffduff-safari
[scriptable]:https://scriptable.app
[workflow]:https://www.workflow.is
[shortcuts]:https://support.apple.com/de-de/guide/shortcuts/welcome/ios
[huffduffer]:https://huffduffer.com
[huffduffit]:https://gist.github.com/schoeffm/fe6a7bdf9af159f6877917c5a51b11aa
[issue]:https://discussions.apple.com/thread/251084795
