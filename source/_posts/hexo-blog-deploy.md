---
title: hexo博客搭建并实现github action自动部署
date: 2021-09-02 07:19:43
tags:
- hexo
- github action
---

> Hexo 是一款基于Node.js的一个快速、简洁且高效的博客框架。Hexo 使用 Markdown（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页，你可以把生成的静态网页上传到 Web 服务器上，这里我们选用的是 GitHub 做 Web 服务器.

<!-- more -->



# 1. 使用 docker 搭建 hexo 博客环境

## 1.1 版本介绍

- hexo: 5.4.0
- hexo-cli: 4.3.0
- node: 16.8.0

## 1.2 部署

```bash
mkdir /data/hexo -p && cd /data/hexo

sudo docker  run -it -d \
	--name hexo \
	-v $PWD:/hexo \
	-p 4000:4000 \
	node:16.8.0 /bin/bash

# 进入容器
sudo docker exec -it hexo /bin/bash

################	下面的操作全是容器中执行	##################
cd /hexo

# 安装 hexo
npm install -g npm@7.21.0
npm install -g hexo-cli
# 新建站点
hexo init blog-markdown

# 添加一篇名叫 welcome 的文章
cd blog-markdown
hexo new welcome

# 编辑文章
# source/_posts/welcome.md

# 编辑完后,生成静态文件
hexo g   # 或 hexo generate

# 发布
hexo server

# 浏览器访问: 
# http://localhost:4000
```

说明:
1. 文章会默认添加到 ` ./source/_posts ` 下.



# 2. 使用 hexo-theme-concise 主题

1. 下载主题

```bash
cd /opt/hexo/blog-markdown
sudo git clone https://github.com/litten/hexo-theme-yilia.git themes/yilia
```

2. 修改 _config.yml 配置: -->  `theme: yilia` 

     <更新主题: `cd themes/yilia; git pull ` >

   

在文件最后增加如下配置:

```yaml
jsonContent:
  meta: false
  pages: false
  posts:
    title: true
    date: true
    path: true
    text: false
    raw: false
    content: false
    slug: false
    updated: false
    comments: false
    link: false
    permalink: false
    excerpt: false
    categories: false
    tags: true

```

3. 切换编译器

   > Hexo 默认的编译器为`hexo-renderer-stylus` 要切换为 `hexo-generator-json-content`

```bash
# 进入到docker: hexo （sudo docker exec -it hexo /bin/bash)  中执行
npm install hexo-generator-json-content --save

# 如果你不需要 hexo-renderer-stylus 可以把它卸载掉
npm uninstall hexo-renderer-stylus --save

# 清除缓存
hexo clean
# 本地查看
hexo server
```



更多进阶操作:

- [Yilia 进阶设置](https://www.gaotianyang.top/archives/20200717e10c0cde/)

- [Yilia主题如何添加相册功能](https://www.gaotianyang.top/archives/20200718f57a1917/)

# 3. 将hexo部署到github



1. 在 github 上创建一个仓库: `yanglibin.github.io`
2. 在你本地使用`ssh-keygen` 命令生成密钥对.
3. 在刚创建的github仓库中设置 ssh 免密:
       `Settings` --> `Deploy keys` --> `Add deploy key` 



4. 安装hexo-deployer-git

```bash
# 进入到 docker:hexo 执行 
npm install hexo-deployer-git
```

5. 修改`_config.yml` 配置

```yaml
deploy:
  type: 'git'
  repo: git@github.com:yanglibin/yanglibin.github.io.git
  branch: master
  # message: [message]  # 自定义提交信息 (默认为 Site updated: {{ now('YYYY-MM-DD HH:mm:ss') }} )
  
```

6. hexo 部署到github

```bash
hexo deploy
```



# 4. 使用github action 自动部署hexo

> 进行此步操作时，你已经完成了 `1.hexo 环境搭建` , `2. 主题配置`



## 4.1 准备github仓库

在 github 上创建一个仓库: `yanglibin.github.io`

- source 分支: 

  > 存放 blog 源码

- master 分支

  > blog 展示



## 4.2 配置密钥

1. 在你本地使用`ssh-keygen` 命令生成密钥对.

2. 在刚创建的github仓库中设置 

- 配置push密钥:(`cat ~/.ssh/id_rsa.pub `)

  `Settings` --> `Deploy keys` --> `Add deploy key` 

- 配置部署密钥:(`cat ~/.ssh/id_rsa`)

   `Settings -> Secrets -> Add a new secret`

  

  在 **Name** 输入框填写 `HEXO_DEPLOY_PRI`, 后面的workflows 中会用到这个变量。 



## 4.3 准备hexo源码

```bash
git clone git@github.com:yanglibin/yanglibin.github.io.git
cd yanglibin.github.io
# 创建并切换到 source 分支
git branch source
git checkout source

# 将 hexo 博客源码 copy 过来
cp -a /opt/hexo/blog-markdown/* ./

# 由于themes/yilia 不会提交, 所以要把一些配置 copy 出来
mkdir -p themes_conf/yilia

cp themes/yilia/_config.yml themes_conf/yilia/

# gitignore
cat > ./.gitignore << EOF
.DS_Store
Thumbs.db
db.json
*.log
node_modules/
themes/
public/
.deploy*/
EOF
```

## 4.4 准备 workflows

```bash
# action workflows
mkdir -p .github/workflows
touch .github/workflows/deploy.yml

# 编写 deploy.yml
# 详见 附录.
```



## 4.5 push 到github

```bash
git add .
git commit -m "my hexo blog first commit"
git push --set-upstream origin source
```



 

# 5. 附录

- [workflows 语法学习](https://docs.github.com/cn/actions/reference/workflow-syntax-for-github-actions)

- [第三方action ](https://github.com/marketplace?type=actions)

- `deploy.yml` 配置示例:

```yaml
# action name
name: Hexo Deploy

# 触发action分支
on:
  push:
    branches:
      - source

# 定义变量
env:
  GIT_USER: yanglibin
  GIT_EMAIL: fengyuanyang000@qq.com
  THEME: yilia
  THEME_REPO: litten/hexo-theme-yilia
  THEME_BRANCH: master
  THEME_PATH: themes/yilia
  NODE_VERSION: 16.8.0


# job
jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      # 使用 第三方 Actions  
      # Checkout 当前仓库到本地
      - name: Checkout
        uses: actions/checkout@v2
        with:
          ref: source


      # Checkout 第三库: https://github.com/litten/hexo-theme-yilia.git 的 master 分支到  themes/yilia 目录.
      - name: Checkout theme repo
        uses: actions/checkout@v2
        with:
          repository: ${{ env.THEME_REPO }}
          ref: ${{ env.THEME_BRANCH }}
          path: ${{ env.THEME_PATH }}


      - name: Set up Node.js
        uses: actions/setup-node@v2
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      # 配置环境
      - name: Configuration environment
        env:
          HEXO_DEPLOY_PRI: ${{secrets.HEXO_DEPLOY_PRI}}

        run: |
          # 配置时区
          sudo timedatectl set-timezone "Asia/Shanghai"
          # 配置ssh 免密
          mkdir -p ~/.ssh/
          chmod 700 ~/.ssh
          echo "$HEXO_DEPLOY_PRI" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan github.com >> ~/.ssh/known_hosts
          # 配置git name, email
          git config --global user.name $GIT_USER
          git config --global user.email $GIT_EMAIL
          # theme
          cp -af themes_config/${{env.THEME}}/*   ${{env.THEME_PATH}}/

      # 配置环境
      - name: install npm dependencies
        run: |
          npm install 
        
      # 生成并部署
      - name: Deploy
        run: |
          npx hexo deploy --generate

```

# 6. 参考链接

- [Hexo 搭建静态博客与 hexo-theme-concise 主题使用教程](https://sanonz.github.io/2017/hello-world/)

- [利用 Github Actions 自动部署 Hexo 博客-1](https://printempw.github.io/use-github-actions-to-deploy-hexo-blog/)
- [利用 Github Actions 自动部署 Hexo 博客-2](https://sanonz.github.io/2020/deploy-a-hexo-blog-from-github-actions/)

