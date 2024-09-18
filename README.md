## gem  bundle  jekyll 三者的关系

### gem
gem 是 Ruby 语言的**包管理工具**, 通过 gem install 命令安装指定的 gem 包
比如gem install jekyll, ruby项目中的gemfile就是依赖的配置文件

### bundle
bundler 是用于 Ruby 项目的**依赖管理工具**，主要通过 Gemfile 文件管理项目所需的 gem。它确保项目在任何环境下使用相同的 gem 版本，避免依赖冲突。
使用 bundle install 命令来安装 Gemfile 中定义的 gem,比如 bundle install

### Jekyll
jekyll 是一个静态网站生成器，是基于 Ruby 的一个 gem。它将 Markdown 文件、YAML 数据等转换为静态的 HTML 网站.
Jekyll 本身是一个 gem，通常通过 gem 或 bundle 安装


```shell
# 创建新的站点 (脚手架)
jekyll new my-site
cd my-site
# 构建项目,安装所需的gems
bundle install
# 启动jekyll
bundle exec jekyll serve
```

### 注意事项
- jekyll 目录结构中,_site是根据markdown文件动态生成,git管理项目时需要排除
- 根目录中的index.html也会在_site中复制一份._site是最终的站点内容
