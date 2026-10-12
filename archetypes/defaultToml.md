+++
title = '{{ replace .File.ContentBaseName "-" " " | title }}'
date = '{{ .Date }}'
categories = []
tags = []
description = 'desc'
draft = true

# cover 作为对象配置
[cover]
    image = 'images/cover.svg'           # 图片路径（必填）
    alt = '山高月小，水落石出。'     # 图片替代文本，利于 SEO 和无障碍访问
    caption = '山水' # 图片标题说明（部分主题支持）
    hidden = false                # 是否在单篇文章页隐藏封面？默认 false
    hiddenInList = false          # 是否在文章列表页隐藏封面？默认 false
    hiddenInSingle = false        # 是否在单篇文章页隐藏封面？(部分主题用此字段)
+++
