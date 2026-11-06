---
title: "{{ replace .File.ContentBaseName "-" " " | title }}"
# slug = final da URL em português (minúsculas, sem acento, com hífens)
slug: "{{ .File.ContentBaseName }}"
translationKey: "{{ .File.ContentBaseName }}"
date: {{ now.Format "2006-01-02" }}
draft: true
tags: []
summary: ""
comments: true
---

Escreva a versão em português aqui.
