---
title: vim配置
date: 2022-07-07 15:03:10
tags:
categories:
- 其他
---

10年前开发ruby时，编辑器是用的vim，当年较真还跟Emacs支持者网上唇枪舌剑，自从转向Java后再也没有使用过。今年希望开始锻炼自己的笔杆子，发现搭建hexo时大量用到vim，不得不开始重新配置vim，顺便流水账似的开始hexo。

## bundle安装

现在都是插件化，vim也是如此，最简单的方式安装vim插件bundle

```bash
git clone https://github.com/VundleVim/Vundle.vim.git ~/.vim/bundle/Vundle.vim
```

## 配置

vim个性化在~/.vimrc文件中配置，包括想用哪些插件，这次我只需要用到nerdtree来提升页面编辑效率。在网上找了些配置方法，但是一直达不到自己习惯的操作，所以还是用了10年前的vim配置来处理。其他有用的插件可以搜索网上的详细方法来配置。具体vimrc配置内容如下：

```
" -----------------  Author: tusyoshi
" -----------------  Date: 2012-10-27
" -----------------  For MAC
set nocompatible              " 去除VI一致性,必须要添加
filetype off                  " 必须要添加

" 设置包括vundle和初始化相关的runtime path
set rtp+=~/.vim/bundle/Vundle.vim
call vundle#begin()

" 让vundle管理插件版本,必须
Plugin 'VundleVim/Vundle.vim'

Plugin 'scrooloose/nerdtree' " 经典nerdtree
Bundle 'Valloric/YouCompleteMe'
Bundle 'scrooloose/syntastic'

" 你的所有插件需要在下面这行之前
call vundle#end()            " 必须
filetype plugin indent on    " 必须 加载vim自带和插件相应的语法和文件类型相关脚本
" 忽视插件改变缩进,可以使用以下替代:
"filetype plugin on
"
" 常用的命令
" :PluginList       - 列出所有已配置的插件
" :PluginInstall  |      - 安装插件,追加 `!` 用以更新或使用 :PluginUpdate
" :PluginSearch foo - 搜索 foo ; 追加 `!` 清除本地缓存
" :PluginClean      - 清除未使用插件,需要确认; 追加 `!` 自动批准移除未使用插件
"
" 查阅 :h vundle 获取更多细节和wiki以及FAQ
" 将你自己对非插件片段放在这行之后
"
"""""""""""""""""""""""""""""""""""""""""""""""""""""
" => 基本配置
"""""""""""""""""""""""""""""""""""""""""""""""""""""
set history=1000   "设置vim history行数
filetype indent on "针对不同的文件类型采用不同的缩进格式
filetype plugin on "针对不同的文件类型加载对应的插件

set autoread "设置自动读取外部修改文件

"设置leader
let mapleader=","
let g:mapleader=","

set ai!                      " 设置自动缩进
set smartindent              " 智能自动缩进
set shiftwidth=2             " 换行时行间交错使用2空格
set cindent shiftwidth=2     " 自动缩进2空格

set list                     " 显示Tab符，使用一高亮竖线代替
set listchars=tab:\|\ ,

set showmatch "显示括号配对情况

"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" => 搜索设置
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
set incsearch                " 开启实时搜索功能
set hlsearch                 " 开启高亮显示结果
set nowrapscan               " 搜索到文件两端时不重新搜索

" 搜索时忽略大小写，但在有一个或以上大写字母时仍大小写敏感
set ignorecase
set smartcase

""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" => 设置window, tab
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" 窗口切换
nnoremap <c-h> <c-w>h
nnoremap <c-l> <c-w>l
nnoremap <c-j> <c-w>j
nnoremap <c-k> <c-w>k

""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" => 设置NERDTREE
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" 默认打开
autocmd vimenter * NERDTree
autocmd VimEnter * wincmd p
autocmd bufenter * if (winnr("$") == 1 && exists("b:NERDTree")) | q | endif
" 让Tree把自己给装饰得多姿多彩漂亮点
let NERDChristmasTree=1
" 窗口宽度
let NERDTreeWinSize=31
let NERDTreeWinPos='left'
map tt :NERDTreeToggle<CR>
" 指定鼠标模式(1.双击打开 2.单目录双文件 3.单击打开)
let NERDTreeMouseMode=2
```

果然是流水账，没点技术含量~
