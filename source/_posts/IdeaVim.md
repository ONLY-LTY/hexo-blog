---
title: IdeaVim
date: 2018-09-26 17:15:28
tags:
---
IdaeVim 配置文件 .ideavimrc
<!--more-->

##### IdaeVim 配置文件 .ideavimrc
```Java
let mapleader=' '
set hlsearch
set incsearch
set ignorecase
set smartcase
set showmode
set number
set scrolloff=8
set history=100000
set clipboard=unnamed

" clear the highlighted search result
nnoremap <Leader>sc :nohlsearch<CR>

nnoremap <Leader>fs :w<CR>

" Quit normal mode
nnoremap <Leader>q  :q<CR>
nnoremap <Leader>Q  :qa!<CR>

" Move half page faster

" Insert mode shortcut
inoremap <C-h> <Left>
inoremap <C-j> <Down>
inoremap <C-k> <Up>
inoremap <C-l> <Right>
inoremap <C-a> <Home>
inoremap <C-e> <End>
inoremap <C-d> <Delete>

" Quit insert mode
inoremap jj <Esc>
inoremap jk <Esc>
inoremap kk <Esc>

" Quit visual mode
vnoremap v <Esc>

" Move to the start of line
nnoremap H ^

" Move to the end of line
nnoremap L $

" Redo
nnoremap U <C-r>

" Yank to the end of line
nnoremap Y y$

" Window operation
nnoremap <Leader>ww <C-W>w
nnoremap <Leader>wd <C-W>c
nnoremap <Leader>wj <C-W>j
nnoremap <Leader>wk <C-W>k
nnoremap <Leader>wh <C-W>h
nnoremap <Leader>wl <C-W>l
nnoremap <Leader>ws <C-W>s
nnoremap <Leader>w- <C-W>s
nnoremap <Leader>wv <C-W>v
nnoremap <Leader>w\| <C-W>v

" Tab operation
nnoremap tn gt
nnoremap tp gT

" ==================================================
" Show all the provided actions via `:actionlist`
" ==================================================

" built in search looks better
nnoremap / :action Find<CR>
" but preserve ideavim search
nnoremap <Leader>/ /

nnoremap <Leader>;; :action CommentByLineComment<CR>

nnoremap <Leader>bb :action ToggleLineBreakpoint<CR>
nnoremap <Leader>br :action ViewBreakpoints<CR>

nnoremap cv :action ChangeView<CR>

nnoremap <Leader>cd :action ChooseDebugConfiguration<CR>

nnoremap ga :action GotoAction<CR>
nnoremap gc :action GotoClass<CR>
nnoremap gd :action GotoDeclaration<CR>
nnoremap gf :action GotoFile<CR>
nnoremap gi :action GotoImplementation<CR>
nnoremap gs :action GotoSymbol<CR>
nnoremap gt :action GotoTest<CR>
nnoremap go :action OptimizeImports<CR>


nnoremap fp :action ShowFilePath<CR>
nnoremap fu :action FindUsages<CR>
nnoremap mv :action ActivateMavenProjectsToolWindow<CR>


nnoremap rc :action ChooseRunConfiguration<CR>
nnoremap re :action RenameElement<CR>
nnoremap rf :action RenameFile<CR>

nnoremap se :action FindInPath<CR>
nnoremap su :action ShowUsages<CR>
nnoremap gb :action Back<CR>
nnoremap gn :action Generate<CR>
nnoremap <Leader>tc  :action CloseActiveTab<CR>
nnoremap <Leader>tl Vy<CR>:action ActivateTerminalToolWindow<CR>
vnoremap <Leader>tl y<CR>:action ActivateTerminalToolWindow<CR>
```

##### VIM 配置文件

```Java
" Configuration file for vim
set modelines=0		" CVE-2007-2438

" Normally we use vim-extensions. If you want true vi-compatibility
" remove change the following statements
set nocompatible	" Use Vim defaults instead of 100% vi compatibility
set backspace=2		" more powerful backspacing

" Don't write backup file if vim is being called by "crontab -e"
au BufWrite /private/tmp/crontab.* set nowritebackup nobackup
" Don't write backup file if vim is being called by "chpass"
au BufWrite /private/etc/pw.* set nowritebackup nobackup

let skip_defaults_vim=1
set nu

set shortmess=atI

syntax on

inoremap jj <Esc>

noremap H ^
noremap L $

set clipboard=unnamed

set scrolloff=5

set nocompatible

set nobackup

set confirm

set mouse=a

set tabstop=4
set shiftwidth=4
set expandtab
set smarttab

set autoread

set cindent

set autoindent

set smartindent

set cursorline

set hlsearch

set background=dark

set showmatch

set ruler

set nocompatible

set fdm=syntax

set fdm=manual

set novisualbell

set laststatus=2

autocmd InsertLeave * se nocul

autocmd InsertEnter * se cul

set showcmd

set fillchars=vert:/

set fillchars=stl:/

set fillchars=stlnc:/

if $TERM_PROGRAM =~ "iTerm"
let &t_SI = "\<Esc>]50;CursorShape=1\x7" " Vertical bar in insert mode
let &t_EI = "\<Esc>]50;CursorShape=0\x7" " Block in normal mode
endif
```
