### About

用途：个人博客，学习记录

[关于](https://www.359sun.top/about/)


### Deploy
install
```shell
npm install hexo-cli -g
hexo init <folder>
cd <folder>
npm install
```

config
```shell
# install next theme
cd <folder>
npm install hexo-theme-next

# vim _config.yml
...
# theme
theme: next

# Deployment
## Docs: https://hexo.io/docs/one-command-deployment
deploy:
  type: git
  repository: git@github.com:yakir3/yakir3.github.io.git
  branch: yakir-blog
```

deploy
```shell
# new post and generate
hexo new "k8s-network"
hexo generate

# start local server
hexo server

# deploy to git
npm install hexo-deployer-git --save
hexo ddeploy
```


>Reference:
> 1. [Hexo Official](https://brew.sh](https://hexo.io/))
> 2. [Hexo Next theme](https://github.com/next-theme/hexo-theme-next)
