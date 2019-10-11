---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "本博客中所有可能会出现的魔法咒语"
subtitle: "Useful Bash Aliases Functions"
summary: "当你在参考本博客的文章时遇到任何你找不到的指令时，这篇文章里的内容或许能帮到你（前提是你没有落下必要的编译安装环节并正确设置系统环境变量）"
authors: ["admin"]
tags: ["alias", "别名", "Bash", "Shell", "环境变量"]
# categories: []
date: 2019-10-11T17:22:24+08:00
# publishDate: the RFC 3339 date that the page was published. You only need to specify this option if you wish to set date in the future but publish the page now, as is the case for publishing a journal article that is to appear in a journal etc.
# lastmod: the RFC 3339 date that the page was last modified. If using Git, enable enableGitInfo in config.toml to have the page modification date automatically updated, rather than manually specifying lastmod.
featured: true
draft: false

# Featured image
# To use, place an image named `featured.jpg/png` in your page's folder.
# Placement options: 1 = Full column width, 2 = Out-set, 3 = Screen-width
# Focal point options: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight
# Set `preview_only` to `true` to just use the image for thumbnails.
image:
  placement: 1
  caption: "Photo by [Artem Maltsev](https://unsplash.com/@art_maltsev?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)"
  focal_point: "Center"
  preview_only: true

# links:
#   - icon_pack: fab
#     icon: medium
#     name: Originally published on Medium
#     url: 'https://medium.com'

reading_time: true  # Show estimated reading time?
share: true  # Show social sharing links?
profile: false  # Show author profile?
commentable: true  # Allow visitors to comment? Supported by the Page, Post, and Docs content types.
editable: false  # Allow visitors to edit the page? Supported by the Page, Post, and Docs content types.

# To enable LaTeX math rendering for a page
# markup: mmark
# math: true

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
# projects: []

---

# 引用说明
以下所列出的许多别名都直接摘自 [Adrien Brochard](http://xmodulo.com/author/adrien) 发表在 [Xmodulo.com](http://xmodulo.com/) 上的[文章](http://xmodulo.com/useful-bash-aliases-functions.html)。感谢他的整理，为我节省了大量的时间。  
我根据我的使用需要修改和添加了一点内容，比如产生不同场合下适用的随机密码、增加对 xz 格式压缩文件的支持，以及快速克隆 GitHub repo 的别名（不要用在其他 Git 存储库上！）等等……  
希望这些别名不仅作为读者理解本站其他博文内容的参考，也能在读者日常使用 Linux 时为您节省时间。

```shell
$ vim ~/.bashrc
```

```shell
alias ls='ls -h --color=auto'
alias ll='ls -alF'
alias la='ls -A'
alias l='ls -CF'

alias grep='grep --color=auto'
alias fgrep='fgrep --color=auto'
alias egrep='egrep --color=auto'

# alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'
alias mkdir='mkdir -p'
alias crontab='crontab -i'

alias histg='history | grep'
alias ps?='ps aux | grep'
alias get_whatever_pw='pwgen -cnsy1 -H <(openssl rand -base64 128)'
alias get_easyread_pw='pwgen -Bcnsy1 -H <(openssl rand -base64 128)'
alias get_easymemo_pw='pwgen -cn1 -H <(openssl rand -base64 128)'

mcd() { mkdir -p "$1"; cd "$1"; }
md5check() { md5sum "$1" | grep "$2"; }
sha1check() { sha1sum "$1" | grep "$2"; }
sha256check() { sha256sum "$1" | grep "$2"; }
sha512check() { sha512sum "$1" | grep "$2"; }

ghclone() {
    mkdir -p /github/
    cd /github/ || exit
    git clone --depth 1 "https://github.com/$1/$2.git"
    cd "$2" || exit
}

extract() {
    if [ -f "$1" ] ; then
      case $1 in
        *.tar.bz2)  tar -xjf "$1"       ;;
        *.tar.xz)   tar -xJf "$1"       ;;
        *.tar.gz)   tar -xzf "$1"       ;;
        *.bz2)      bunzip2 "$1"        ;;
        *.rar)      unrar e "$1"        ;;
        *.gz)       gunzip "$1"         ;;
        *.tar)      tar -xf "$1"        ;;
        *.tbz2)     tar -xjf "$1"       ;;
        *.txz)      tar -xJf "$1"       ;;
        *.tgz)      tar -xzf "$1"       ;;
        *.zip)      unzip "$1"          ;;
        *.Z)        uncompress "$1"     ;;
        *.7z)       7zr x "$1"          ;;
        *)          echo "'$1' cannot be extracted via extract()"   ;;
      esac
    else
        echo "'$1' is not a valid file"
    fi
}
```
