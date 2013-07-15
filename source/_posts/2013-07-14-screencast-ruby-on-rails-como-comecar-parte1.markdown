---
published: true
layout: post
title: "Screencast: Ruby on Rails - How to Learn? - part 1"
date: 2013-07-14 22:46
comments: true
categories:
 - Rails
---

<iframe width="560" height="315" src="//www.youtube.com/embed/3TYrOp-6ltw" frameborder="0" allowfullscreen></iframe>

<!-- more -->

[rubyonrails.org](http://rubyonrails.org) (!?)

Get Started
-----------
  * Agile Web Dev book: Loja Virtual (Depot)
  * Rails Tutorial Book: Microblog (TDD / daily work / best practices)

1) Install Ruby via Rbenv!
--------------------------
```
sudo apt-get install git

git clone https://github.com/sstephenson/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bash_profile
echo 'eval "$(rbenv init -)"' >> ~/.bash_profile
exec $SHELL -l

git clone https://github.com/sstephenson/ruby-build.git ~/.rbenv/plugins/ruby-build
rbenv install --list

sudo apt-get install make
rbenv install 2.0.0-p247
```

2) Install Rails via rubygems
-----------------------------
```
gem install rails
rbenv rehash
```

3) New project
--------------
```
sudo apt-get install sqlite3 libsqlite3-dev nodejs # Database and JavaScript runtime

rails new loja_virtual
rails s
open http://localhost:3000
```

4) Version Control with Git
---------------------------
```
git init
git add .
git commit -m "Initial commit"
```

5) Github Social Coding
-----------------------
Make development more collaborative
```
git remote add origin git@github.com:<username>/<projectname>.git

# generate keypair - only if you don't have one
cd ~/.ssh
ssh-keygen -t rsa -C "your_email@example.com"

# copy this command's output to your Github SSH Keys management page
cat id_rsa.pub

git push -u origin master
```

Next Stop
---------
Deploy (!!!) to Amazon Web Services Free Tier
