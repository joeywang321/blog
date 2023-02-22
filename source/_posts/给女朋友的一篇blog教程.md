---
title: 给女朋友的一篇blog教程
date: 2023-02-22 10:59:06
tags:
---

# hexo搭建blog

保姆教程，耐心观看，主要是写给我的乖乖的。

## 本地搭建

安装 hexo，使用 npm 安装，所以需要本地先安装 nodejs（详情观看 node 官方文档）
```bash
npm install hexo-cli -g
```

在 shell 里面切换到自己想存放 blog 文件的路径，macos 的文档下或者 windows 的 D 盘，自己选择。
myblog 是文件名，可以写成自己想要的文件名，执行完后就会在你选择的文件路径下生成一个 myblog 文件夹。
```bash
hexo init myblog
```

cd 进入你的文件夹然后执行命令安装依赖
```bash
cd myblog && npm install
```

这样在不安装主题的情况下，我们的本地就算搭建完成了。
现在我们可以执行命令在本地打开我们的博客，看看效果。
```bash
hexo s
```

注意： 主题的安装和配置详情参见 hexo theme 根据自己的喜爱进行配置

## github

现在开始创建 github repository，在 github 创建一个名为 myblog 的仓库（可以自定义 name）
在本地 shell 进入你的 blog 路径，并进行 git 初始化
```bash
cd myblog && git init
```

添加一个 commit ，并提交到你的 github 仓库（基础操作就不详细描述了）大概是是下面的流程
```bash
git add .
git commit -m 'feat: xxxxx'
git branch -M main
git remote add origin your_repository
git push -u origin main
```

## server ssh key

我们 actions 使用的是 ssh 协议进行文件传输，需要生成 ssh key （跳转到 actions 部分观看）

进入 .ssh  文件夹，并生成ssh key
```bash
cd ~/.ssh
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

**注意：我们在生成SSH Key的时候，不能使用[Github的generating an SSH key page上的默认说明](https://docs.github.com/en/github/authenticating-to-github/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)。这是因为 Github Actions 不支持最新的 Ed22159 算法。我们需要改用 legacy 命令。**

接下来我们需要命名 SSH 密钥文件。在这里，我不建议使用默认文件名（即 id_rsa ）。建议将文件名切换为， github-actions 以便我们知道此密钥用于 Github Actions。（在你执行生成命令后就会让你输入你的文件名，不输入就代表使用默认的 id_rsa ）

**注意：我们还被要求提供密码。将此留空，因为当 Github Actions 为我们运行 SSH 命令时我们无法输入密码。**

现在你执行ls命令应该会看到生成两个文件 github-actions 和 github-actions.pub
```bash
ls
github-actions github-actions.pub authorized_keys
```

将公钥添加到 authorized_keys

最简单的方法是使用 cat 命令追加 github-actions.pub 到 authorized_keys 

```bash
cat github-actions.pub >> ~/.ssh/authorized_keys
```

**注意：确保使用双尖角括号( >> )而不是单尖括号 ( > )。double 表示附加，而 single 表示覆盖!**

## github actions

下面就是关于上传github的actions的完整版，我们会分开慢慢讲。

```yaml
name: build and deploy

on:
  push:
    branches:
    - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout Code
      uses: actions/checkout@v3

    - name: Install Dependencies
      uses: actions/setup-node@v3
      with:
        node-version: 16
    - run: npm install hexo-cli -g
    - run: npm install

    - name: Build
      run: hexo clean && hexo generate

    - name: Deploy
      uses: easingthemes/ssh-deploy@main
      env:
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
          SOURCE: "./public/"
          REMOTE_HOST: ${{ secrets.REMOTE_HOST }}
          REMOTE_USER: ${{ secrets.REMOTE_USER }}
          TARGET: ${{ secrets.REMOTE_TARGET }}
```

触发条件：当 push 到 main 分支时开始执行此 CI
```yaml
name: build and deploy

on:
  push:
    branches:
    - main
```

运行环境：使用 ubuntu 作为系统环境

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
```

加载代码

```yaml
steps:
  - name: Checkout Code
    uses: actions/checkout@v3
```

安装依赖：安装 node 和 hexo
```yaml
- name: Install Dependencies
  uses: actions/setup-node@v3
  with:
    node-version: 16
- run: npm install hexo-cli -g
- run: npm install
```

构建

```yaml
- name: Build
  run: hexo clean && hexo generate
```

上传部署

我们使用的是 ssh 协议进行上传部署所以需要 ssh key
```yaml
- name: Deploy
  uses: easingthemes/ssh-deploy@main
  env:
      SH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
      SOURCE: "./public/"
      REMOTE_HOST: ${{ secrets.REMOTE_HOST }}
      REMOTE_USER: ${{ secrets.REMOTE_USER }}
      TARGET: ${{ secrets.REMOTE_TARGET }}
```

在你的 github myblog 仓库的 settings 中找到 secrets/actions 进行设置上面所需要的

SSH_PRIVATE_KEY  →  ~/.ssh/github-actions  注：不是 github-actions.pub

REMOTE_HOST → 服务器的公网 ip

REMOTE_USER → 服务器的用户一般是 root

REMOTE_TARGET → 文件会上传服务器这个路径下，需要在服务器先创建这个文件夹

## nginx

nginx 的部署配置文件，初学者也可以使用宝塔进行部署

```yaml
server {
    listen 80;
    server_name your_domain_name;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name your_domain_name;

    # SSL证书和私钥文件的路径
    ssl_certificate /path/to/cert.crt;
    ssl_certificate_key /path/to/key.pem;

    # 加密算法的优先级
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384;

    # 配置根目录 根据自己的目录来
    root /var/www/blog;

    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```
