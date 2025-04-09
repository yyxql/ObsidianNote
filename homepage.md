```dataviewjs
(async () => {
  // 递归生成目录导航的函数
  async function listRecursive(folder = "", depth = 0) {
    let output = "";
	const vaultName=app.vault.adapter.basePath.split(`\\`).slice(-1);
    const indentPx = depth * 10; // 每一层级的缩进

    // 获取当前文件夹下的所有文件和子文件夹
    let listResult;
    try {
	    listResult = await app.vault.adapter.list(folder);
    } catch (e) {
	    listResult = { files: [], folders: [] };
    }

    // 获取当前文件夹中的 Markdown 文件
    const pages = dv.pages(`"${folder}"`);
    const filesInCurrentFolder = pages.where(p => (p.file.folder || "") === folder);
    
    if (folder) {
      // 使用 <details> 和 <summary> 创建可折叠的目录项
		output += `<details style="margin-left: ${indentPx}px;" >\n  <summary>${folder.split("/").pop() || "Root"}</summary>\n  <ul>\n`;
    } else {
	    output += `<ul>\n`;
    }
    
    // 列出当前文件夹下的文件
    filesInCurrentFolder.forEach(file => {
	    // output += `    <li style="margin-left: ${indentPx + 20}px;" >${file.file.link}</li>\n`;
		output += `<li style="margin-left: ${indentPx + 20}px;"><a href="obsidian://open?vault=${encodeURIComponent(vaultName)}&file=${encodeURIComponent(file.file.path)}">${file.file.name}</a></li>`;
    });

    let sortedSubfolders = listResult.folders
      .filter(subfolder => {
        // 获取文件夹的最后一部分，并过滤以 “.” 开头的目录
        const baseName = subfolder.split("/").pop();
        return !baseName.startsWith(".");
      })
      .sort((a, b) => a.localeCompare(b, 'zh')); 
    
    // 递归处理子文件夹
    for (let subfolder of sortedSubfolders) {
	    output += await listRecursive(subfolder, depth + 1);
    } 

    if (folder) {
	    output += `  </ul>\n</details>\n`;
    } else {
	    output += `</ul>\n`;
	}

    return output;
  }

  // --- 生成首页内容 ---

  // 1. 目录导航
  dv.header(2, "目录导航");
  const directoryNavigation = await listRecursive();
  dv.paragraph(directoryNavigation);

  // 2. 最近更新的文章
  const recentFiles = dv.pages("")
    .where(p => !p.file.path.includes("Templates"))
    .sort(p => p.file.mtime, 'desc')
    .limit(5);

  dv.header(2, "🕒 最近更新");
  dv.table(["文件", "修改时间", "创建时间"],
    recentFiles.map(p => [
      p.file.link,
      p.file.mtime.toFormat("yyyy-MM-dd HH:mm"),
      p.file.ctime.toFormat("yyyy-MM-dd")
    ])
  );

  // 3. 未完成的任务
  const tasks = dv.pages("").file.tasks
    .where(t => !t.completed)
    .sort((t1, t2) => {
      if (t1.due && t2.due) {
        return t1.due - t2.due;
      } else if (t1.due) {
        return -1;
      } else if (t2.due) {
        return 1;
      }
      return 0;
    });

  dv.header(2, "📌 待办事项");
  dv.taskList(tasks, false);

  // 可选：添加额外的 CSS 样式以优化导航列表的外观
  dv.el("style", `
    details {
      margin-bottom: 5px;
    }
    details > summary {
      cursor: pointer;
      list-style: none;
    }
    ul {
      list-style-type: none;
      padding-left: 0;
    }
  `);
})();
```
