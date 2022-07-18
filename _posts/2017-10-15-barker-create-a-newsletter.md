---
toc: false
layout: post
categories: [python]
title: How I made my own newsletter from my bookmarks with Python
---

I have a lot of bookmarks that I actively curate. I’m kinda a bookmark junkie, carefully considering where each one belongs. For example, here’s a picture of what my visualization folder looks like:

![]({{ site.baseurl }}/images/Screen Shot 2017-11-26 at 2.31.00 PM.png "Bookmarks")

All these bookmarks come from great newsletters such as [Python weekly](https://www.pythonweekly.com/), [Data Elixir](https://dataelixir.com/), [Data Science Weekly](https://www.datascienceweekly.org/) and others. So I thought what would be more natural than creating a newsletter out of my bookmarks!

That’s when [Barker](https://github.com/cheevahagadog/barker) was born. Set to a cron job, you can create your own newsletter for yourself, or customize it a bit to be for your company. Here’s an example of one of mine:


![]({{ site.baseurl }}/images/Screen Shot 2017-10-05 at 2.50.36 PM.png "Bookmarks")
![]({{ site.baseurl }}/images/Screen Shot 2017-10-05 at 2.50.42 PM.png "Bookmarks")
![]({{ site.baseurl }}/images/Screen Shot 2017-10-05 at 2.50.52 PM.png "Bookmarks")

You can configure it so it will take X number of recent bookmarks, have it focus on only one folder, use all except a few, and filter out bookmarks in a given domain (such as stackoverflow or a Google doc).

Currently it only works with Chrome, though I’ll look into adding support for other browsers if there is any interest.

To get started using it, check it out here https://github.com/cheevahagadog/barker. If you use it and have any suggestions, I’d be glad to hear them! Feel free to submit a pull request, open an issue, or ping me on social media.
