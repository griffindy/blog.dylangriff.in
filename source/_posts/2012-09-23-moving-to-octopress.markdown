---
layout: post
title: "Moving to Octopress"
date: 2012-09-23 22:34
comments: true
categories: [code, ruby, octopress, jekyll]
---

I used to host my blog on Tumblr because that seemed like an easy solution. But
starting today I'm moving to [Octopress](http://octopress.org/) because I
actually think it's even easier to use. Octopress fits better into my blog
eriting workflow and appeals to my geeky side because I'm doing all of the
hosting/deploying/what-have-you myself.
<!--more-->

How easy is using Octopress? As easy as downloading the source and running `rake
install`. After doing and `rake preview` you'll see the standard Octopress theme
in use widely across the internet. After creating a new post (which I'll cover
in a minute), you run `rake generate` and `rake deploy` and your blog has been
pushed to the world wide web. That being said, there were a few hiccups I found
that I can hopefully save someone else from having.

First, if you're using ZSH, the standard syntax for creating a new post (which
is actually the stand syntax for sending a Raketask an argument), won't work.
The command is supposed to read `rake new_post['My Awesome Post']`, but ZSH
tries to glob with the brackets. The solution is to use `noglob rake
new_post['My Awesome Post']`. You can either just remember to do this or put an
alias in your `.zshrc`. The other thing that I found tricky was where to put a
custom font I had downloaded ([League
Gothic](http://www.theleagueofmoveabletype.com/league-gothic)) in order for it
to be compiled with everything else. For some reason my first step was not
`assets`, but that is where they belong.

I wrote earlier that Octopress fits into my workflow better, and that is because
I like to write in vim as much as possible. I could certainly write in vim and
then copy and paste my writing into my tumblr, but that was always much clunkier
if I ever had to edit something. Editing an Octopress post merely involves
finding it in `source/_posts` and then `rake generate`. However, I think my
favorite part of Octopress is `rake deploy`. Octopress understands a few
different methods of deployment (like Github Pages and Heroku), but the most
basic, and the one I use, is [rsync](http://linux.die.net/man/1/rsync) because I
already have a [Linode](http://www.linode.com). rsync is a very fast file
copying tool that saves a lot of time by *not* copying files that haven't been
changed. There is no git, and since Octopress uses
[Jekyll](https://github.com/mojombo/jekyll) everything is technically static
content, there is no need to muck around with different servers like Thin,
Unicorn or Passenger, just good ole' Nginx or Apache.

So far I'm loving Octopress, and I hope you are too.
