+++
date = "2017-12-09T03:57:28+02:00"
draft = false
title = "New setup for this blog: Migration from Ghost to Hugo"

tags = [ "Blogging platform" ]
+++
### World Wild Jungle
As I already complained in the post [Reporting: Data Lab Web App](/the-data-lab-web-app/) earlier this year, I am quite confused regarding the actual complexity of web development.  The evolution of the web technologies created a brand new world.  World Wild virtual jungle. The diversity of species living there is gigantic. It is getting closer to real nature where it is impossible for a man to get detail understanding of all living forms. Even the scientists have to specialize and focus on specific subset only.  

We live in the times of rapid expansions of web technologies. New forms arise each day while others are reaching their dead end.  So to pick just one particular platform to fit perfectly your needs in long-term is definitely a challenge.  Especially when running a personal blog is not a full-time job but only a hobby. 

### Evolution path of my blogging platform
Don't get me wrong, there is a plenty of friendly platforms which you can instantly use for easy blogging.  The problems start when you want to be in control of things.  When you want to experiment with any part of a content and implement whatever idea you get.

Then you need some good open source platform with a strong community around.  

Okay, I  will tell you now my own evolution path. First I got a suggestion from my friend to join the [Drupal](https://www.drupal.org/) community which is "of course" the best. But I quickly realized that to cope with such monster is far beyond my capacity. Then I picked the [Django](https://www.djangoproject.com/), because I like Python and it looked so much easier.  It is definitely much easier than Drupal but still, I  didn't get there to the point when I can focus more on what I want than how to achieve it. The next step was the [Ghost](https://ghost.org/).  With Ghost, I finally started this blog and soon realized what I really need. Simplicity and freedom!  But the Ghost gives you just the first. 

Then I heard about the [Hugo](https://gohugo.io/). Well, the first impression wasn't that great. Static site generator, why to go for anything like that?  Fortunately, I was already playing with the idea of static pages for some time. And in fact, there is not much sense in storing the web content in a database to later extract it specifically for each user.  I understand, there are some super cash mechanisms to avoid performance disaster but isn't that too complicated for delivering written texts? If the motivation is teamwork and collaboration isn't then better to use the best tool for that allowing not only team collaboration but also complete versioning control? 

### The final compilation
Questions like that lead to a combination of versioning tool and static site generator.  The generator takes care of easy maintenance, separation of content from its format. It deals with all the dependencies and libraries providing optimal frontend framework. This gives you a perfect compilation of tools where you don't need to dive into details before you really need to customize something.  

And in case of customization, you only focus one thing at one time.  Sure you have to go out of the comfort zone where everything is fixed by somebody else but you are not paying any extra costs for going on your own. Not like with the other monster tools where any minor change requires first building huge complicated infrastructure which allows you to change some tiny thing. Here you simply add your own isolated piece of work on the top of what is already created. There are no limits to that and you don't even pay any extra tax for that.

Sounds great? It is great.  The migration of my prior posts from Ghost to Hugo was super easy. It even uses the same Markdown syntax and shares a similar approach for storing static files like images. In fact, the most difficult part of the migration was to accept the fact that the solution is that simple. Exactly like this post, the most of the talking is just to swallow that fact. 

### Step by step  implementation
Now the shorter part describing the implementation begins.  The final solution uses [Git](https://git-scm.com/) as collaboration and versioning tool, [GitHub](https://github.com/) as project repository and also website hosting provider and finally [Cloudflare](https://www.cloudflare.com/) services to Set Up SSL on Github Pages With Custom Domains. All that for free. 

<div class="c_alert c_alert-success"><i class="fa fa-exclamation-triangle"></i> Prerequisites:  installed Git and created GitHub account</div>

<b>Step 1</b>: [Install Hugo](https://gohugo.io/getting-started/installing), create new website 
``` 
hugo new site your_site_name
``` 
pick some [Hugo theme](https://themes.gohugo.io/) and copy its directory into the website themes directory. Copy the content of exampleSite directory from the theme directory to the website root directory.

<b>Step 2</b>: [Create new GitHub project repository](https://github.com/new) for your web, clone it on your pc 
``` 
git clone https://github.com/your_user/your_project.git
``` 
and move there the complete content of your Hugo website root directory. Check the structure for [this blog](https://github.com/lubomirkamensky/lubolab) if it's not clear.

<b>Step 3</b>: Update config.toml, setting publishDir = "docs", test the site locally
``` 
hugo server -D
``` 
publish the content for the web.
``` 
hugo
``` 
if not clear, check [quick-start manual](https://gohugo.io/getting-started/quick-start/).

<b>Step 4</b>: Push all the changes to your GitHub repository.
``` 
git add --all :/
git commit -m "Meaningful commit description" 
git push  
``` 
<b>Step 5</b>: Go to GitHub repository settings, section GitHUb Pages  and set the source as master branch/docs and set  your domain as the Custom domain.

<b>Step 6</b>: Pull the changes to your local repository
``` 
git pull
``` 
copy CNAME file from DOCS directory to your static  directory so it is not lost in case you drop and recreate docs directory. You can check the final structure on GitHub for [this blog](https://github.com/lubomirkamensky/lubolab).

<b>Step 7</b>: Update DNS records for your domain, create two A records pointing to 192.30.252.153 and 192.30.252.154, create CNAME record pointing www to your_domain.com.  Use Cloudflare.com to [Set Up SSL on Github Pages With Custom Domains](https://hackernoon.com/set-up-ssl-on-github-pages-with-custom-domains-for-free-a576bdf51bc)

