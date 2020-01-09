---
title: "Experiments"
date: 2020-01-08T20:36:36+01:00
draft: false
comments: true
---

I've always found well-written essays particularly fascinating, to the point that I've decided to experiment on finding
a method to convey  messages effectively that suits me. In order to feel committed, I purchased a domain on [Google Domains](https://domains.google)
(took 5 minutes in total and €15) and hosted this basic website. I have always used [R](https://www.r-project.org) at work and I am planning to transition
from sales/acc. management roles to more analytical ones, This space will be used to collect gotchas and observations I find interesting through this path.
 Landing on Hugo as a blogging platform has been somewhat of a natural decision:

* stupidly easy to use (just choose a theme and write your posts in markdown, tweaking theme's .css file as needed)
* deployed on github and hosted for free on [netlify](https://netlify.com) (5 minutes in total)
* good syntax highlighting: this will be crucial as I'll be mainly writing about R and analytics in general

## Getting started

Setting everything up has taken an hour at most. First of all, purchase a domain from whatever registrar you choose
(I've chosen [Google Domains](https://domains.google) to avoid registering and verifying a new account).
Below the steps I've followed to install hugo on my Linux machine (I went for the 'longish' route as `sudo apt-get install hugo` was
installing an old version which gave problem with the theme I chose). All the steps listed below are explained at length on [Hugo docs](https://gohugo.io/documentation/).  

To follow the steps above you need to download the right tarfile from your system from Hugo's [github repo](https://github.com/gohugoio/hugo/releases)
```sh
# Move downloaded tarball and extract it 
sudo mv ~/Donwloads/hugo_0.62.2_Linux-64bit.tar.gz /usr/local/bin/
sudo tar -xvzf /usr/local/bin/hugo_0.62.2_Linux-64bit.tar.gz

# Check everything has worked
➜  ~ hugo version
Hugo Static Site Generator v0.62.2-83E50184 linux/amd64 BuildDate: 2020-01-05T18:51:38Z
```

Now we can initialise our website and choose a theme (I went for [hugo-notepadium](https://github.com/cntrump/hugo-notepadium) for its simplicity):

```sh
# Create new website
cd ~/code
hugo new site your_website_name

# Add hugo theme
cd your_website_name
git init
git submodule add https://github.com/cntrump/hugo-notepadium themes/hugo-notepadium

# Push your website to a github repo
git remote add origin https://github.com/your-username/your-repo.git
git push -u origin master
```
We can check everything has worked properly by launching our website locally with `hugo server`.  
Depending on your theme, you can customize your website by editing fields in `config.toml` file, this allows to setup things like:

* general theme appearance
* title and website base url
* Google Analytics `tracking-ID`

Creating content in Hugo is as simple as launching `hugo new my_first_post.md` and editing the newly created markdown file.

## Hosting

I am currently experimenting with GCP, but the steps to host a simple static website in a cloud Storage Bucket were kind of overwhelming for a beginner
(putting a load balancer in front of the bucket to get https certificates working). Because of that, I've decided to opt for netlify's free plan instead.
Hosting a website is as simple as:

* Opening an account
* Pointing to the github repo created above steps
* Setting up a custom domain by pointing to the right DNS. In my case this involved setting up the following entries in Google Domains settings (under DNS tab):  

|Name|Type|TTL|Data|
|----|:--:|:-:|:---:|
|@|A|1H|104.198.14.52|
|www|CNAME|1H|your_netlify_project_code|

It took around ~24h for the settings to propagate, but after that everything will be up and running.
