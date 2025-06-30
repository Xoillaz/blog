---
title: 笔记｜hi, hexo
comments: false
date: 2025-06-29 20:59:06
updated:
categories:
tags: SOP
---
# 我的写作流
{% note info %}
如果不把内容和样式分离的话，一边写作一边调整格式会分散注意力，建议写好了再慢慢调整。
{% endnote %}

1. 打磨内容
   1. 随时随地都可以写作，完成后将初稿同步到 source/_drafts
   2. 将初稿新建为 draft，继续进行图片引用等工作；
2. 丰富样式
   1. 将稿件新建为 post，启动 Hexo 实时预览；
   2. 在 VSCode 一边预览网页一边调整格式；
3. 发布文章：推送到 Github，自动部署到 Github Page
- 依赖 
  1. VSCode 的简单浏览器：`Crtl+Shift+V -> Simple Browser: Show`
  2. VSCode 拓展：Markdown All in One

{% grouppicture 2-2 %}
{% asset_img edit-before-publish.png before %}
{% asset_img edit-after-publish.png after %}
{% endgrouppicture %}

# 我的博客部署过程
1. [安装 hexo 框架及 NeXT 主题](https://theme-next.js.org/docs/getting-started/)
2. [按需修改 _config.next.yml，配置主题](https://theme-next.js.org/docs/theme-settings/)
3. [采用 Github Page 的方式部署网站](https://hexo.io/zh-cn/docs/github-pages)

{% note info %}
**在部署网站时需要注意**
{% tabs 部署网站注意事项 %}
<!-- tab 修改模板 yml -->
须确认发布分支、替换 Node.js 为本地安装的实际版本

{% codeblock .github/workflow/hexo.yml lang:yaml %}
name: Deploy Hexo site to Pages

on:
  push:
    branches: main

...

      - name: Use Node.js 22.x  
        uses: actions/setup-node@v4
        with:
          node-version: "22"

...
{% endcodeblock %}
<!-- endtab -->

<!-- tab 确保部分文件不会被上传到远程仓库 -->
例如，我的目录结构如下：

{% codeblock 本地目录 https://github.com/Xoillaz/blog 我的远端目录 %}
.  
├── .gitignore        # git 在推送代码时应忽略的文件列表
├── .gitmodules       # git 子模块配置
├── _config.yml       # hexo 默认配置
├── _config.next.yml  # NeXT 主题配置
├── package.json      # hexo 自动生成的依赖声明
├── package-lock.json # hexo 自动生成的依赖快照
├── .github           # Github Actions 配置
├── node-modules          # 本地测试环境，不应该被同步到远端仓库
├── public                # 本地预览时生成的静态网站文件，不应该被同步到远端仓库
├── scaffolds         # hexo 模板
├── source            # hexo 资源
|   ├── _drafts           # 未发布在网站上的文章，不应该被同步到远端仓库
|   └── _posts        # 已发布在网站上的文章
└── themes            # hexo 主题
{% endcodeblock %}

其中有一些文件不应该被同步到远端仓库，因此需要在 `.gitignore` 文件追加以下内容。

{% codeblock .gitignore %}
node_modules/
public/
source/_drafts/
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}
{% endnote %}

# 值得一提的标签语法
[**mermaid**](https://theme-next.js.org/docs/tag-plugins/mermaid)：代码即图表
{% tabs mermaid snippet %}
<!-- tab 流程图 -->
```
{% mermaid graph TD %}
A[Hard] -->|Text| B(Round)
B --> C{Decision}
C -->|One| D[Result 1]
C -->|Two| E[Result 2]
{% endmermaid %}
```
<!-- endtab -->
<!-- tab 饼图 -->
```
{% mermaid pie %}
"fish": 1
"dog": 3
{% endmermaid %}
```
<!-- endtab -->
<!-- tab 甘特图 -->
```
{% mermaid gantt %}
dateFormat  YYYY-MM-DD
section Section
Completed :done,    des1, 2014-01-06,2014-01-08
Active        :active,  des2, 2014-01-07, 3d
Parallel 1   :         des3, after des1, 1d
Parallel 2   :         des4, after des1, 1d
Parallel 3   :         des5, after des3, 1d
Parallel 4   :         des6, after des4, 1d
{% endmermaid %}
```
<!-- endtab -->
<!-- tab 状态图 -->
```
{% mermaid stateDiagram %}
[*] --> Still
Still --> [*]
Still --> Moving
Moving --> Still
Moving --> Crash
Crash --> [*]
{% endmermaid %}
```
<!-- endtab -->
<!-- tab 类图 -->
```
{% mermaid classDiagram %}
Class01 <|-- AveryLongClass : Cool
<<interface>> Class01
Class09 --> C2 : Where am i?
Class09 --* C3
Class09 --|> Class07
Class07 : equals()
Class07 : Object[] elementData
Class01 : size()
Class01 : int chimp
Class01 : int gorilla
class Class10 {
  <<service>>
  int id
  size()
}
{% endmermaid %}
```
<!-- endtab -->
{% endtabs %}

**其它插件**
{% tabs 标签插件介绍, -1 %}
<!-- tab tabs -->
如果我只用传统的 markdown 语法写作，在呈现方式上水平空间没有被很好地利用到。[tabs](https://theme-next.js.org/docs/tag-plugins/tabs) 标签弥补了这点。

```
{% subtabs example %}
<!-- tab 1 -->
clock
<!-- endtab -->
<!-- tab 2 -->
york
<!-- endtab -->
<!-- tab 3 -->
step
<!-- endtab -->
{% endsubtabs %}
```

{% subtabs example %}
<!-- tab 1 -->
clock
<!-- endtab -->
<!-- tab 2 -->
york
<!-- endtab -->
<!-- tab 3 -->
step
<!-- endtab -->
{% endsubtabs %}
<!-- endtab -->

<!-- tab note -->
我在表达的时候很喜欢用括号补充信息，[note](https://theme-next.js.org/docs/tag-plugins/note) 标签可以让我更直观地传达心意。

```
{% note info %}
#### 关于 xoillaz 与 xorillaz
xorillaz 是我原来的英文名，改为 xoillaz 只是因为这样更好看。我觉得 xoillaz 读起来像 choiless，嗯～ choice less，别无选择那就静下心来好好干吧。
{% endnote %}
```

{% note info %}
**关于 xoillaz 与 xorillaz**
xorillaz 是我原来的英文名，改为 xoillaz 只是因为这样更好看。我觉得 xoillaz 读起来像 choiless，嗯～ choice less，别无选择那就静下心来好好干吧。
{% endnote %}
<!-- endtab -->

<!-- tab codeblock -->
[codeblock](https://hexo.io/docs/syntax-highlight#How-to-use-code-block-in-posts) 标签在文章插入代码时可以指定语言类型和文件名等选项。

```
{% codeblock example.sh lang:bash %}
sudo su
{% endcodeblock %}
```

{% codeblock example lang:bash %}
sudo su
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}