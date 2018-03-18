---
title: "{{ replace .Name "_" " " | replaceRE "^\\d+-(.+)$" "$1" | title }}"
date: {{ .Date }}
draft: false
tags: []
---
