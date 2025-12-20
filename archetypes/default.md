+++
date = '{{ .Date }}'
draft = true
title = '{{ replace .File.ContentBaseName "-" " " | title }}'

# SEO / Social
# - PaperMod uses .Description for meta description + OG/Twitter/Schema. If omitted, it falls back to .Summary.
description = ''
tags = []
categories = []
keywords = []

# PaperMod reads .Params.cover.image for:
# - post cover rendering
# - OG/Twitter image (highest priority)
[cover]
image = ''
alt = ''
caption = ''
relative = false
hiddenInSingle = true
+++
