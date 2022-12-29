---
title: "Beginner Git Commands"
layout: post
name: "git-beginner-commands"
tags: ["Git"]
date: 2022-11-24 00:00:00 +0100
author: "Darren Foley"
show_sidebar: false
menubar: cheatsheet_menu
---

Beginner git cheatsheet covering commands for working with local repositories in addition to working with remote repositories. Details on working with Github are included below as it is by far the most popular amongst developers. 

**Setting up a local repository**

```
> cd mydir
> git init
```

This is a once off command that initializes a .git sub-directory in the current working directory mydir.

<br>

**Cloning an existing repository from github/remote**

There are two clone protocols; SSH or HTTPS. Use HTTPS if you have not added your public key to github (see below).


```
> git clone https://github.com/<github username>/<repo name>.git
```

Assuming you have set up SSH you can use

```
> git clone git@github.com:<github username>/<repo name>.git
```

<br>

**Adding an SSH key to Github**
 
If you wish to authenticate with github using SSH you will need to add your SSH public key (generated locally) to github.

Follow the instructions on github [here](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account).

<br>

**Using Pesonal Access (PAT) Tokens**

If you do not wish to authenticate using username/password or SSH, github provides Personal access tokens to authenticate with a remote repo. Follow the instructions [here](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) on how to set that up.

<br>


**Setting local git configuration**

Git stores configuration at /.git/config (repository) or /.gitconfig (User) or /etc/gitconfig (Global).


For global settings use the --global flag.

```
> git config --global user.name "John Snow"
> git config --global user.email "john.snow@gmail.com"
```

Use --local to change just the current repository settings.


```
> git config --local user.name "John Snow"
> git config --local user.email "john.snow@gmail.com"
```

<br>

**Adding Changes to the staging area**

Git has three main areas for files; current working directory, staging area and git database. Use the staging area for breaking up a large change into multiple commits. 

```
> git add file1 file2 file3
```


