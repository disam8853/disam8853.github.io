---
title: '在 Github Page 上安裝免費個人部落格 Jekyll'
date: 2021-06-19
categories:
  - blog
tags:
  - Jekyll
  - github page
  - blog
  - free
---

> 本編將講述如何使用 `Jekyll` 在 Github Page 上來建立專屬個人的 Blog 網站。

# 1. 前言

Github Page 提供開發者一個免費的架站方法，但是有個限制是必須只能是靜態網頁，無法開啟 server，因此另一套自動產生靜態網頁的工具 **Jekyll** 就誕生了，他解決了 Github Page 無法建立動態網站的限制，讓使用者可以輕鬆地建立自己的部落格。

# 2. 工具

- Github 帳號

# 3. 安裝

## 3.1. step 1: install bundler

因為 Jekyll 是使用 ruby 開發，因此需要在自己電腦安裝 ruby 的套件管理工具 [Bundler](https://bundler.io/)，

## 3.2. step 2: 選擇一個好看的 Jekyll 主題

Jekyll 有許多各式各樣的主題可以選擇，可以在以下網站查看所有主題：

- [Github](https://github.com/topics/jekyll-theme)
- [jamstackthemes.dev](https://jamstackthemes.dev/ssg/jekyll/)
- [jekyllthemes.org](http://jekyllthemes.org/)
- [jekyllthemes.io](https://jekyllthemes.io/)
- [jekyll-themes.com](https://jekyll-themes.com/)

筆者在這裡選擇 Github 中 stars 數量最多的 [minimal-mistakes](https://github.com/mmistakes/minimal-mistakes) 作為示範（也是這個部落格所使用的主題）。

## 3.3. step 3: 安裝至 Github

選好主題之後就可以按照他的指示安裝到自己的 Github 帳號下的 repo 。

根據 [minimal-mistakes](https://github.com/mmistakes/minimal-mistakes) 指示，我選擇使用 **Remote theme method** 來安裝，打開這個[網址](https://github.com/mmistakes/mm-github-pages-starter/generate)之後就並且輸入 repo name: USERNAME.github.io，USERNAME 換成自己的 Github 帳號即可，比如說我的 repo name 就會是 diasm8853.github.io，這是 Github Page 規定的 repo name，如果不照這個格式命名就會失敗。

## 3.4. step 4: 修改 config

建立好 repo 之後就可以用 git clone 下來並且可以開始客製化專屬自己的部落格。

clone 下來之後需要先進行一次 `bundle install` 安裝需要的套件：

```bash
cd USERNAME.github.io
bundle install
```

想要在 local 測試的話就可以輸入：

```bash
bundle exec jekyll serve
```

預設 server 會在 localhost:4000

# 4. 撰寫文章

在 `_posts/` 裡可以加入自己寫的新文章，命名規則為 `YYYY_MM_DD-article-title.md`，比如說本篇的檔案名稱即為 `2021-06-19-setup-customized-blog-at-github-page.md`。
