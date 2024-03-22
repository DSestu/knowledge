---
layout: default
title: Misc
nav_order: 1
parent: Windows
---

### Unquoted service paths

`cmd /c wmic service get name,displayname,pathname,startmode |findstr /i "auto" |findstr /i /v "c:\windows\\" |findstr /i /v """`