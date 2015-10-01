title: hexo and github page
date: 2015-10-01 16:44:08
tags:
---

### 利用github page 和 hexo 搭建自己的博客
###### 主要记下自己的排雷过程

##### github page
注册[gitbug](https://github.com/)账号，创建一个名字为username.github.io的新仓库，配置[ssh](https://help.github.com/articles/generating-ssh-keys/)

#### hexo
安装[hexo](https://hexo.io/docs/)
``` bash
$ hexo init blog
$ cd blog
$ npm install
```
在Hexo 3.0版本后还需要单独安装hexo git，安装命令为: npm install hexo-deployer-git --save ,安装完成后执行会在当前目录下生成blog的文件夹，此目录即为你的博客生成基地了。
由于hexo与传统的博客托管网站不同的一点是博客的源文件是保存在本地的，
为了以后能够方便的在多台终端上使用，我们可以把源文件同步到github仓库中的新建分支中， 执行如下命令，把源文件提交到 username.github.io的source（新chekout的分支）仓库中
``` bash
$ cd blog
$ git init
$ git checkout -b source
$ git add .
$ git commit -m "first commit"
$ git remote add origin git@github.com:username:username.github.io.git
$ git push origin source
``` 

修改全局配置文件 _config.yml， 找到[deploy]()修改
deploy:
  type: git
  repository: git@github.com:username/username.github.io.git
  branch: master

##### 工作就绪，开始写博客
``` bash
hexo new “New Post"（文章的title）
``` 
会在source/_posts下新建New Post.md的文件，编辑该文件，
执行
``` bash
hexo d -g
``` 
发布到github page 上

##### 在浏览器中 输入username.github.io 查看