#Nian Tianlei' Blog

###[View DEMO &rarr;](https://niantianlei.github.io)  

![](http://p1.bpimg.com/567571/83bd22fcc93e0262.png)  



##  fork  

If you want to clone my blog? Here comes this!    

```
$ git clone git@github.com:niantianlei/niantianlei.github.io.git  
```

##### New Feature  

- **[Comment](#comment)** support [**duoshuo**](http://duoshuo.com/) officially.  


-  **[Analytics](#analytics)** is configurable now!   





- Brand new **[Keynote Layout](#keynote-layout)** is provided for easily posting beautiful HTML presentations you have created with this blog




## Support

- **Feel free to fork. I'll Appreciate it if you keep the Author & Github link at footer**
- Give it a **Star** if you like, fork or just clone to use ;)
- If any problem or requirement, just open an issue here and I will help you.


## Document

* Get Started
	* [Environment](#environment)
	* [Get Started](#get-started)
	* [Write Posts](#write-posts)
* Components
	* [SideBar](#sidebar)
	* [Mini About Me](#mini-about-me)
	* [Featured Tags](#featured-tags)
	* [Friends](#friends)
	* [Keynote Layout](#keynote-layout)
* Comment & Analysis
	* [Comment](#comment)
	* [Analytics](#analytics)

#### Environment

If you have jekyll installed, simply run `jekyll serve` in Command Line
and preview the themes in your browser. You can use `jekyll serve --watch` to watch for changes in the source files as well.


#### Get Started

You can easily get started by modifying `_config.yml`:

```
# Site settings
title: Nian Tianlei' Blog             # title of your website
SEOTitle: Nian Tianlei' Blog   	      # check out docs for more detail
description: "Cool Blog"              # ...

# SNS settings      
github_username: niantianlei          # modify this account to yours
weibo_username: niantianlei           # the footer woule be auto-updated.

# Build settings
# paginate: 10                        # nums of posts in one page
```

There are more options you can check out in the [Jekyll - Official Site](http://jekyllrb.com/), or you can directly dive into code to find more.


#### Write Posts

Feel free to checkout Markdown files in the `_posts/`, you will quickly realized how to post your articles with magical markdown plus this nice theme.

The **front-matter** of a post looks like that:

```
---
layout:     post
title:      "Hello 2016"
subtitle:   "Hello World, Hello Blog"
date:       2016-12-5 12:00:00
author:     "Nian"
header-img: "img/post-bg-2016.jpg"
tags:
    - Life
---

```

#### SideBar


Seeing more information may be necessary for you to display, a clean, gorgeous **SideBar** is added for you, which provide more area for displaying possible modules. You can enable *(it is default enable)* this feature by simply config:

```
# Sidebar settings
sidebar: true
sidebar-about-description: "your description here"
sidebar-avatar: /img/at.jpg     # use absolute URL.
```

We default support *[Featured Tags](#featured-tags)*, *[Mini About Me](#mini-about-me)* and *[Friends](#friends)* these three modules and you can add your own. The sidebar is naturally responsive and would be push to bottom in a small screen size (`<= 992px`, according to [Bootstarp Grid System](http://getbootstrap.com/css/#grid))  
More details of these three separate modules are talking below.

#### Mini About Me

Mini-About-Me module display all your SNS buttons also your avatar and the description if you set `sidebar-avatar` and `sidebar-about-description` which is very useful and common for a sidebar so it is default with your sidebar.

It is really nice-looking and well-designed. It would be hidden in a small screen seeing the sidebar would be push to bottom and there is already a footer including SNS feature which is similar.

#### Featured Tags

Considering the Featured-Tags feature in [Medium](http://medium.com) is pretty cool, so I add it in my blog theme also.   
This module is independent of sidebar from V1.4, so it can definitely live without enable sidebar, which would be displayed in the bottom when `sidebar` set to false, and it is not only displayed in home page but also every post page bottom.


```
# Featured Tags
featured-tags: true  
featured-condition-size: 1     # A tag will be featured if the size of it is more than this condition value
```

The only one thing need to be paid attention to is the `featured-condition-size`: A tag will be featured if the size of it is more than this condition value.  
Internally, a condition template `{% if tag[1].size > {{site.featured-condition-size}} %}` is used to do the filter.

#### Friends

Friends is a very common feature of a blog seeing the SEO, so I add it in V1.1 release to help that.   
Friends can also live without enable sidebar, also be displayed in the bottom when sidebar unable, and be displayed in every post page bottom.


You can just add your friends information in `_config.yml` with a familiar JSON syntax and everything is done, very easy:

```
# Friends
friends: [
    {
        title: "Foo Blog",
        href: "http://foo.github.io/"
    },
    {
        title: "Bar Blog",
        href: "http://bar.github.io"
    }
]
```


#### Keynote Layout


There is a increasing tendency to use Open Web technology to create keynotes, presentations, like Reveal.js, Impress.js, Slides, Prezi etc. I consider a modern blog should have abilities to post these HTML based presentation easily also abilities to play it directly.

Under the hood, a `iframe` is used to include webpage from outer source, so the only things left is to give a url in the **front-matter**:

```
---
layout:     keynote
iframe:     ""
---
```

The iframe will be automatically resized to adapt different form factors also the device orientation. A padding is left to imply user that there has more content below, also to ensure that there is a area for user to scroll down in mobile device seeing most of the keynote framework prevent the browser default scroll behavior.


#### Comment

This theme support both [Disqus](http://disqus.com) and [Duoshuo](http://duoshuo.com) as the third party discussion system.

First, you need to sign up and get your own account. **Repeat, DO NOT use mine!** (I have set Trusted Domains) It is deathly simple to sign up and you will get the full power of management system. Please give it a try!

Second, from V1.5, you can easily complete your comment configuration by just adding your **short name** into `_config.yml`:

```
duoshuo_username: niantianlei  
# OR
disqus_username: _your_disqus_short_name_
```

**To the old version user**, it's better that you pull the new version, otherwise you have to replace code in `post.html`, `keynote.html` and `about.html` by yourselves.

Furthermore, Duoshuo support Sharing. if you only wanna use Duoshuo comment without sharing, you can set `duoshuo_share: false`. You can use Duoshuo Sharing and Disqus Comments together also.



#### Analytics

# Baidu Analytics
ba_track_id: 06960db8bccee304d53fdb570602d0e4


Just checkout the code offered by Baidu, and copy paste here, all the rest is already done for you.




## License


My Blog is derived from [Clean Blog Jekyll Theme (MIT License)](https://github.com/BlackrockDigital/startbootstrap-clean-blog-jekyll/)

