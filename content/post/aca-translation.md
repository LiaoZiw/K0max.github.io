+++
title = '论文翻译版和原版合成同页对比版pdf'
date = 2024-06-07T23:48:45+08:00
draft = false
+++

# 为啥要整这个

我在使用学术 GPT 进行原文翻译的时候, 发现中文版偶尔会有乱码, 而且是凭空占一页(把该页正常的翻译内容挤到下一页去了), 这样导致对比版内容对不上.

# 如何解决?

我用了比较笨的办法, 把单独的翻译版 pdf 拿出来, 手动删除错误页, 然后手动把原版和翻译版拼接起来, 形成像学术 GPT 那样的 comparison版本, 但是怎么做到呢? 我用了一点 LaTeX 的技术

```tex
\documentclass{article}
\usepackage[utf8]{inputenc}
\usepackage{pdfpages}
\usepackage{forloop}

\begin{document}
\newcounter{x}
\forloop{x}{1}{\value{x} < 23}{
  \includepdfmerge[nup=1x2, landscape]{RCNN-en.pdf, \arabic{x}, RCNN-zh.pdf, \arabic{x}}
}
\forloop{x}{23}{\value{x} < 26}{
  \includepdfmerge[nup=1x2, landscape]{RCNN-en.pdf, \arabic{x}}
}

\end{document}
```

主要就是使用 pdfpages 包, 英文版有 25 页, 中文版有 22 页, 第一个 loop 就是一页一页放上去, 第二个 loop 是把英文版剩下的 3 页放上去

