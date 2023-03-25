---
layout: post
title: "transplant pixyll theme on MPE"
subtitle: "介绍pixyll移植方法"
date: 2021-12-06
author: "Cheney.Yin"
header-img: "img/bg-material.jpg"
tags:
 - Typora
 - Vscode
 - Markdown Preview Enhanced
 - theme
---

# 为MPE移植pixyll主题

最近**Typora**发布了正式版，然而**Typora**开启了收费模式。于是决定在**vscode**上配置新的`markdown`环境。

**Typora**拥有出色的预览能力，**vscode**平台上的**MPE**（Markdown Preview Enhanced）刚好满足需要。但是，**MPE**的默认主题并不包含`pixyll`。所以尝试将**Typora**的`pixyll`移植到**MPE**。

**MPE**默认提供了`github.css`、`newsprint.css`等主题，根据这些线索可以确定**MPE**插件的部署位置。

```shell
find ~/. -name 'newsprint.css'
/home/cheney/./.vscode/extensions/shd101wyy.markdown-preview-enhanced-0.6.1/node_modules/@shd101wyy/mume/styles/preview_theme/newsprint.css
/home/cheney/./.config/Typora/themes/newsprint.css
```
通过上述命令，可以发现MPE的主题部署在了`/home/cheney/./.vscode/extensions/shd101wyy.markdown-preview-enhanced-0.6.1/node_modules/@shd101wyy/mume/styles/preview_theme/`目录下。

然后，将Typora的pixyll.css主题文件复制到上述目录，

```shell
cp -r ~/.config/Typroa/themes/pixyll.css ~/.config/Typora/themes/pixyll ~/.vscode/extensions/shd101wyy.markdown-preview-enhanced-0.6.1/node_modules/@shd101wyy/mume/styles/preview_theme/
```

完成复制后，尝试修改**vscode**的`setting.json`文件，在文件中添加**MPE**的主题配置。

```json
{
    ...

    "markdown-preview-enhanced.previewTheme": "pixyll.css"
    ...
}
```

通过预览**Markdown**文件，可以发现预览主题已经修改为`pixyll`。但是，**vscode**会提示错误。

```json
Value is not accepted. Valid values: "atom-dark.css", "atom-light.css", "atom-material.css", "github-dark.css", "github-light.css", "gothic.css", "medium.css", "monokai.css", "newsprint.css", "night.css", "none.css", "one-dark.css", "one-light.css", "solarized-dark.css", "solarized-light.css", "vue.css".
```

这是因为自定义主题不在`markdown-preview-enhanced.previewTheme`规范内。

对刚才提到的问题，需要修改**MPE**插件的`package.json`文件，向`markdown-preview-enhanced.previewTheme`的`enum`内追加`pixyll.css`即可。