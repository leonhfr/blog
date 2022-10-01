---
title: {{ replace .Name "-" " " | title }}
description: Description
date: {{ .Date | time.Format "2006-01-02" }}
author: leon h
draft: true
favorite: false
# tags:
#   - tag1
---

