# 部署指南

按照以下步骤将你的博客部署到GitHub Pages：

## 1. 创建GitHub仓库

1. 登录你的GitHub账号
2. 创建一个新的仓库，命名为 `username.github.io`（用你的GitHub用户名替换username）
3. 将仓库设为公开

## 2. 将本地代码推送到GitHub

```bash
# 进入publish目录
cd publish

# 初始化Git仓库
git init

# 添加所有文件到暂存区
git add .

# 提交更改
git commit -m "Initial commit"

# 添加远程仓库
git remote add origin https://github.com/username/username.github.io.git

# 推送到GitHub
git push -u origin main
```

## 3. 设置GitHub Pages

1. 在你的GitHub仓库页面，点击"Settings"
2. 找到"Pages"选项（在左侧菜单）
3. 在"Source"部分，选择"main"分支
4. 点击"Save"
5. 稍等片刻，你的网站将可以通过 `https://username.github.io` 访问

## 4. 使用自定义域名（可选）

如果你有自己的域名，可以按照以下步骤配置：

1. 在你的DNS服务商处添加以下DNS记录：
   - A记录：指向185.199.108.153
   - A记录：指向185.199.109.153
   - A记录：指向185.199.110.153
   - A记录：指向185.199.111.153
   - CNAME记录：将www子域名指向 `username.github.io`

2. 在GitHub仓库的"Settings" > "Pages"页面，在"Custom domain"字段中输入你的域名
3. 点击"Save"
4. 勾选"Enforce HTTPS"（推荐）

## 5. 本地预览

在推送到GitHub之前，你可以在本地预览你的博客：

```bash
# 安装依赖
bundle install

# 启动本地服务器
bundle exec jekyll serve
```

然后在浏览器中访问 `http://localhost:4000` 即可预览。 