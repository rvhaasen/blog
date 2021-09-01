---
title: "Creating a website with Hugo"
date: 2021-01-11T16:00:09Z
image: hugo-logo.png
draft: false
---
There are many ways to create simple good-looking websites for personal usage.
Until recently I put my tech-blogging in Confluence ( which is the main documentation system at my work).
Work-related info should ofcourse be shielded from the outside, 
non-confidential subjects ('hobby projects') could however be stored somewhere else.
I bumped into 'Hugo' and immediatly got enthousiastic about the concept.
The result is the blogging site that you are looking at.  

## Whats so nice about Hugo?

I like hugo because:

- **I'm not a web-developer and want to focus on content** \
In Hugo, content is written in mark-down.
- **The software is opensource**
- **Hugo builds a static website** \
Hugo translates the resources into static html. This is easy to deploy to a webserver. Also there are much less security concerns (no databases that can be compromised or otherwise accessed) 
- **Result can directly be viewed during development** \
For this, hugo can serve the site localy and update the result when content is added or changed. This results is a very smooth experience: just update the content from e.g. vim and immediately see the results in the browser! 
- **Hugo is fast**

## Cookbook
There are many resources on the web how Hugo can be used.
I started with following video. There are some problems when following this method, these are addressed in my 'cookbook' section. In the video, Hugo is installed on the host. I perfer to run it from docker, this is also described in the cookbook.
(Update: after some time I changed theme from 'Casper' to 'Mainroad'', the reason is that Casper is not updated for newer hugo version causing things to break)
{{< youtube c7vpcqA6SEQ>}}

After several days of exploration I got a good grip on it. There were several issues that had to be solved. These will be addressed in following sections.
Here my cookboot how to use Hugo on Windows (same method can be used on Mac and Linux):

1. **Install Docker**   
  In this way, there will be no pollution on the host. On Windows, Docker-desktop is an easy way to install docker.

2. **Open a shell** 
  cd to where you want to store in localy.

3.  **Start the Hugo container and get into a shell** 
    ```bash
    $ cd c:\projects
    $ mkdir blog
    $ docker run --name hugo --rm -it -p 1313:1313 -v ${pwd}:/src 
        klakegg/hugo:0.82.0-alpine shell 
    hugo:/src$ 
    ```
4. **Initialize a new site 'blog'** 
    ```bash
    hugo:/src$ hugo new site blog 
    hugo:/src$ cd blog
    ```
5.  **Install a theme**   
    ( I used 'casper-two')
    ```bash
    hugo:/src$ cd themes 
    hugo:/src$ git clone <your selected theme>
    ``` 
6.  **Copy the config file from the example**   
    Check the example configuration file 'config.toml' in the installed theme. Copy it to the root of the new project. 
7.  **Run hugo**  
   in order to view the site on http://localhost:1313
    ```bash
    hugo:/src$ hugo serve -D
    ```
8. **Check the site**   
    use the video to check things if the site does not render as expected.
9. **Create a second terminal connection to the container**  
    (In this way, Hugo stays active in the other shell instance)
    ```bash
    $ cd c:\projects\blog
    $ docker exec -it hugo sh
    ```
10. **Create a first blog entry**  
     ```bash
     $ hugo new post/my-first-post.md
     ```
     Check the video for details. Note that the blog will not be visible from the home-page. It is visible when clicking on 'posts'  



## When the magic fails...
### General
1.  **Emoji's are not rendered by default**  
    add following line to the configuration:   
    ```
    enableEmoji = true  
    ```
### Specific for the casper-theme
1.  **As mentioned, the latest blog is not visible on the main page**  
    The reason is because Hugo has 
been updated after the video, this causes an issue the casper theme (author should update the theme code...) \
In order to solve it, change following line in the file 'themes\casper-two\layout\partials\post-list.html' \
    from:  
    ```
    {{ $paginator := .Paginate (where .Data.Pages.ByDate.Reverse "Type" "post") }}
    ``` 
    to:
    ```
    {{ $paginator := .Paginate (where .Site.RegularPages.ByDate.Reverse "Type" "post") }}
    ```
    Now the latest blog should be visible on the main page

2.  **No date visible in blog list**  
    The used theme 'casper-two' does not seem to have a date displayed in the listing. For this add following line in the same file as above:
    change:
    ```
    <h2 class="post-card-title">{{.Title}}</h2>
    ``` 
    into:
    ```
    {{ .Date.Format "January 2, 2006" }}
    <h2 class="post-card-title">{{.Title}}</h2>
    ``` 
    Now the dates should be displayed above the tiles (in blog list mode)

3.  **Code-blocks not rendered well on mobile devices**  
    Appearently the codeblocks do not adapt to the screen, and long lines cause problems. Best to split long lines into shorter ones and test it using developer tool ( in browser)





## Manage and host the code in Github
After the site is running fine localy, let hugo create the static site by running "hugo" (parameter publishdir in config determines the output location of the generated site). The resulting files can be copied to a webserver.
Github offers a very elegant and easy way to manage the your code and host it on your github account. If you don't want to have your code in the public, you can use a private repositories. \
Check [here](https://gohugo.io/hosting-and-deployment/hosting-on-github/) for instructions. 

## Final thoughts
This posting should give a complete working example how to create a site with Hugo, using Casper-two as theme. 
Although you don't need html knowledge, it does help to solve issues which you might encounter. The developer tool in your browser is also very usefull. Having said that, once the initial hurdles are taken, is fun to update and maintain your site with Hugo!
