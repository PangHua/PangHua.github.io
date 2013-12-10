---
layout: post
title: Python vim env
---

{{ page.title }}
================

<p class="meta">10 Dec 2012 - BeiJing Ring Building</p>

# Install ctags
    
    # yum install -y ctags

# Generate tags
    
    # cd /usr/lib/python2.6/site-packages/neutron/
    # ctags -R

# Check whether python.vim exists
    
    # find / -name "python.vim"
    /usr/share/vim/vim72/ftplugin/python.vim
    /usr/share/vim/vim72/syntax/python.vim
    /usr/share/vim/vim72/indent/python.vim

# Configure vim 
    # cat ~/.vimrc
    syntax on
    filetype indent plugin on
    set ruler
    set showcmd
    set completeopt+=longest
    set modeline
    set background=dark
    set filetype=python
    au BufNewFile,BufRead *.py,*.pyw setf python
    set autoindent " same level indent
    set smartindent
    set expandtab
    set tabstop=4
    set shiftwidth=4
    set softtabstop=4
    set tags=/usr/lib/python2.6/site-packages/neutron/tags,/usr/lib/python2.6/site-packages/neutron/

#  Valid file .vimrc
    
    # source ~/.vimrc

# Configure bashrc

    # cat ~/.bashrc
    alias vi='vim'

Refference:
    
<http://linux-wiki.cn/wiki/zh-hans/%E9%85%8D%E7%BD%AE%E5%9F%BA%E4%BA%8EVim%E7%9A%84Python%E7%BC%96%E7%A8%8B%E7%8E%AF%E5%A2%83>
<http://linux-wiki.cn/wiki/%E7%94%A8Vim%E7%BC%96%E7%A8%8B%E2%80%94%E2%80%94%E9%85%8D%E7%BD%AE%E4%B8%8E%E6%8A%80%E5%B7%A7>
<http://www.cnblogs.com/samwei/archive/2011/04/25/2026211.html>
    
