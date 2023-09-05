---
title: Welcome to My Obsidian Blog
Folder: content
---
目录
```dataview
list title
from "diaoyb.github.io/content" 
WHERE !icontains(file.name,"index")
sort file.ctime desc
```

```
<%*
const dv = app.plugins.plugins["dataview"].api;
const weekGoals = await dv.queryMarkdown(`list title
From "diaoyb. Github. Io/content" 
WHERE !Icontains (file. Name,"index")
Sort file. Ctime desc`);
tR += weekGoals.value;
%>
```

