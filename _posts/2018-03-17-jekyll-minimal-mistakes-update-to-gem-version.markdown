---
title: "Jekyll: Updating and migrating minimal mistakes to gem version"
layout: single
date:   2018-03-17 09:27:08 +0100
excerpt: "Updating minimal mistakes to the latest version by using the gem version instead of manual installation"
classes: wide
categories:
  - Jekyll
tags:
  - jekyll
  - minimal-mistakes
---
As not much happened with my blog in the past months, there have been no updates to
the software used. So the last version was built using some old versions of jekyll
and the minimal mistakes theme which i had manually installed. At the time getting
the comments working with staticman was a bit of a challenge.

Next challenge now: Update jekyll and minimal mistakes. To make things easier in
the future i want to use the gem version of the minimal mistakes theme. This is how
i did it:

## 1. Update jekyll to the newest version (on my Mac)

I used the [following gist](https://gist.github.com/vookimedlo/4276b93fe3b29e85fa4df9b8865edc37#file-jekyll-installation-macos-md) which explains the installation process nicely.

## 2. Create a Jekyll site for the gemified install

````
jekyll new blog3.faerber.me
````

## 3. Review and update the Jekyll configuration

Review and change the newly created configuration (_config.yml) to meet your needs
and also include the gemified version of minimal mistakes:

* Change directory to your newly created site directory
````
cd blog3.faerber.me
```
* Download, review and amend _config.yml (based on minimal mistakes [default config](https://github.com/mmistakes/minimal-mistakes/blob/master/_config.yml))
````
curl -o _config.yml https://raw.githubusercontent.com/mmistakes/minimal-mistakes/master/_config.yml
````
* Add "theme: minimal-mistakes-jekyll" to "_config.yml"
* Add "gem "minimal-mistakes-jekyll"" to "Gemfile" (Take note of the quotes)
* Add ""jekyll-archives"" to "Gemfile" in the jekyll-plugins section

## 4. Remove unnecessary files and download default minimal mistakes files

* Remove index.md, about.md, _posts/*welcome-to-jekyll*
````
rm index.md about.md _posts/*welcome-to-jekyll*
````
* Download minimal mistakes default index.html
````
curl -o index.html https://raw.githubusercontent.com/mmistakes/minimal-mistakes/master/index.html
````

## 5. Re-enable Archives (i had that enabled, links to the files i use)

````
mkdir _pages _data
curl -o _pages/category-archive.html https://gist.githubusercontent.com/andreasfaerber/f82e79343a7ab825e351118bf7571c73/raw/d686f1bad32cd9c52866f9def23e3d5b4c83fb1f/2018-03-jekyll_mmistakes_category-archive.html
curl -o _pages/year-archive.html https://gist.githubusercontent.com/andreasfaerber/3dc7deaef2ee45f9a519d1b61ad1260b/raw/975fff130e867e07f02807143c37ad1bda0ecf58/2018-03-jekyll_mmistakes_year-archive.html
curl -o _data/navigation.yml https://gist.githubusercontent.com/andreasfaerber/379fb8defadfa9ef8e49d8ab5d18a1a5/raw/5a1b6842aa91b902d13ec16ed5901a7babbfc63a/2018-03-jekyll_mmistakes_navigation.yml
````

## 6. Copy _posts and _drafts over

Copy the posts (_posts directory) and drafts (_drafts) directory over from your old site.
Also copy anything else (like _pages) that you created previously (about.md?).

That's it. I have not yet had a look at static comments via staticman. Will check that
up in a bit. As i only have one comment for now, i'll delay that for a small time.
