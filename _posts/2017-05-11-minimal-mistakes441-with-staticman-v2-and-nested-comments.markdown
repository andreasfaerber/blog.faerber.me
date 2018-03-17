---
title: "Minimal Mistakes: v4.4.1 with staticman v2 and nested comments"
layout: single
date:   2017-05-11 20:22:28 +0100
excerpt: "Minimal mistakes nested comments: How to implement staticman v2 and nested comments for the Jekyll Minimal Mistakes theme v4.4.1"
classes: wide
categories:
  - Jekyll
tags:
  - minimal-mistakes
  - staticman
 
---
# Introduction
Prior to starting my (still quite young) blog i had to decide which software to choose
for running the blog. The idea of static pages was quite convincing, in particular in
combination with using staticman for comments.

As i chose [minimal-mistakes](https://github.com/mmistakes/minimal-mistakes)
as theme for my blog, staticman (v1) integration came out of the box.

The nested comment feature as implemented by [Michael Rose](https://mademistakes.com)
(the author of the minimal-mistakes theme) felt convincing and so i tried to implement
it. Often. Failed often. And i failed more often as Javascript, HTML and various other
things utilized by the theme are far away from my comfort zone.

In the end i was able to make it after i understood some of the relationship between
the templates (comment.html, comments.html), javascript (fill form fields dynamically)
and stylesheets (appearance and "formatting").

That felt rather bothersome and i got some kind of training in doing the changes
repeatedly, i try to help others not to loose that much time and hair while trying
to do so. After doing so i created a patch between the initial repository and the
"final" repository which has nested comments with staticman v2 working.

So far i have not easily been able to get mail notifications and reCaptcha working
smoothly and thus they are currently disabled. Will work on that every now and then
to get this incorporated, too.

For anyone interested: I basically copied the relevant files from Michaels jekyll site over and amended things where required.

Note: I didn't get the lazyload stuff working which Michael Rose uses in for his
blog so i included [lazysizes](https://github.com/aFarkas/lazysizes) within the patch. 

This is how you could quickly to where i got, too:

# 1. Fork the minimal-mistakes repository

Fork the minimal-mistakes repository into your github account [Click here to fork](https://github.com/mmistakes/minimal-mistakes/fork). For this blog entry i renamed the created
for to "mmistakes441-staticmanv2".

# 2. Add staticman as a collaborator

Add staticman as a collaborator for your forked repository. See the [minimal-mistakes documentation](https://mmistakes.github.io/minimal-mistakes/docs/configuration/) and search for "Add Staticman as a Collaborator". Call the api endpoint of staticman v2 afterwards:

https://api.staticman.net/v2/connect/{ your github username }/mmistakes441-staticmanv2

It is important to use the v2 API (see v2 in the link).

# 3. Get the diff

You can either clone the [minimal-mistakes-441-to-staticman-v2 repository](https://github.com/andreasfaerber/minimal-mistakes-441-to-staticman-v2) or [download the diff
directly](https://raw.githubusercontent.com/andreasfaerber/minimal-mistakes-441-to-staticman-v2/master/mmistakes441-staticmanv2-nested-comments.diff).

# 4. Clone your repository

Clone your freshly forked minimal-mistakes repository and cd into the directory.

{% highlight shell %}
afaerber@a:~/src$ git clone git@github.com:andreasfaerber/mmistakes441-staticmanv2.git
Cloning into 'mmistakes441-staticmanv2'...
remote: Counting objects: 9684, done.
remote: Total 9684 (delta 0), reused 0 (delta 0), pack-reused 9684
Receiving objects: 100% (9684/9684), 31.86 MiB | 4.33 MiB/s, done.
Resolving deltas: 100% (5392/5392), done.
Checking connectivity... done.
afaerber@a:~/src$ cd mmistakes441-staticmanv2
afaerber@a:~/src/mmistakes441-staticmanv2$
{% endhighlight %}

# 5. Apply the patch

Apply the diff downloaded or cloned before. I cloned it parallel to the mmistakes441-staticmanv2 directory.

{% highlight shell %}
afaerber@a:~/src/mmistakes441-staticmanv2$ git apply --reject ../minimal-mistakes-441-to-staticman-v2/mmistakes441-staticmanv2-nested-comments.diff
{% endhighlight %}

# 6. Re-generate Javascript files

Regenerate the Javascript files into the main.min.js file by running the
following shell script created by the patch:

{% highlight shell %}
afaerber@a:~/src/mmistakes441-staticmanv2$ sh ./do_uglify.sh
afaerber@a:~/src/mmistakes441-staticmanv2$
{% endhighlight %}

# 7. Fill in your Github data

Edit "do_update_github.sh" which has also been created via the patch and fill in
your GITHUB_USERNAME, GITHUB_REPOSITORY and GITHUB_BRANCH (which will stay at master if you followed the steps here).

{% highlight shell %}
GITHUB_USERNAME=andreasfaerber
GITHUB_REPOSITORY=mmistakes441-staticmanv2
GITHUB_BRANCH=master
{% endhighlight %}

Execute it:
{% highlight shell %}
afaerber@a:~/src/mmistakes441-staticmanv2$ sh ./do_update_github.sh
afaerber@a:~/src/mmistakes441-staticmanv2$ 
{% endhighlight %}

# 8. Push changes to Github

Commit and push the changes to Github:

{% highlight shell %}
git add .
git commit -m 'Updated to staticman v2'
git push
{% endhighlight %} 

# 9. Jekyll serve

Build and serve your Jekyll 
{% highlight shell %}
afaerber@a:~/src/mmistakes441-staticmanv2$ bundle exec jekyll serve
Configuration file: /home/afaerber/src/mmistakes441-staticmanv2/_config.yml
Configuration file: /home/afaerber/src/mmistakes441-staticmanv2/_config.yml
            Source: /home/afaerber/src/mmistakes441-staticmanv2
       Destination: /home/afaerber/src/mmistakes441-staticmanv2/_site
 Incremental build: disabled. Enable with --incremental
      Generating...
   GitHub Metadata: No GitHub API authentication could be found. Some fields may be missing or have incorrect data.
                    done in 5.755 seconds.
 Auto-regeneration: enabled for '/home/afaerber/src/mmistakes441-staticmanv2'
Configuration file: /home/afaerber/src/mmistakes441-staticmanv2/_config.yml
    Server address: http://127.0.0.1:4000/
  Server running... press ctrl-c to stop.
{% endhighlight %}

# 9. Testing

Browse to <http://127.0.0.1/> and check if comments and nested comments are working:

Upon submitting a comment you see a new pull request in your Github repository. Merge that
pull request, run "git pull" in your repository and either run Jekyll serve again
or if it's running in the background, wait for it to re-generate your site.

Reload the page.


Let me know if it also works for you or about issues you found.
