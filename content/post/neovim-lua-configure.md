---
title: "配置Neovim作为Markdown编辑器"
date: 2023-05-17T20:50:45+08:00
draft: true
tags: ['neovim']
---

由于要和`Obsidian`工作流联动，而`Obsidian`本身的编辑能力实在是远逊于`vim`，
再在`Obsidian`环境下配置一遍`vim-mode`大可不必，所以正好重新梳理一遍`Neovim`的[配置](https://git.azurcrystal.com/AzurCrystal/nvim)。

这次采用[Lazy.Nvim](https://github.com/folke/lazy.nvim)作为插件管理器，将原先的`init.vim`用`lua`重写了。

<!--more-->

> **Warning**  
> 以下配置均在快速迭代之中，仅作参考。
> 如有需求，请使用[LazyVim](https://github.com/LazyVim/LazyVim)。

> **Info**  
> **人使用工具，而非工具使用人。**  
> 请牢记这一点。

基于现有的`vim`插件选择，我非常推荐以下两种方案：

- `Vim-Plug` + `Coc-Nvim`  
  非常经典而且传统的配置方案，开箱即用。
  只需要下载少量`coc`插件即能自行运作，接近`vscode`的体验。  
  缺点是加载速度慢，无法拆分文件。

  
- `LazyVim`  
  基于`lazy.nvim`的纯`lua`插件解决方案。
  有非常现代化的UI预设和工具预设。
  新增配置成本很大，但是高效成熟，易于管理，
  有插件延迟加载功能。

如果之前没有`vim`相关使用经历，而且不需要与命令行打多少交道的话，  
**请使用`vscode`。**  真的。用`vscode`吧。


  
