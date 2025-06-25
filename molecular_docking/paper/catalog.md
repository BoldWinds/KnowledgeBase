```dataviewjs
// --- 配置区 ---
// 定义你想要在表格中显示的属性及其对应的表头名称
const columns = {
  "file.link": "笔记标题",
  "Description": "描述",
  "Publication Year": "年份",
  "Rating": "评级",
  "Status": "状态",
  "Tag": "标签",
  "DOI": "DOI"
};
// --- 配置区结束 ---

// 获取当前笔记所在的文件夹路径
const currentFolder = dv.current().file.folder;

// 查找当前文件夹（及其子文件夹）下的所有笔记
// 并排除当前这个目录笔记自身
const pages = dv.pages(`"${currentFolder}"`)
    .where(p => p.file.path !== dv.current().file.path);

// 按文件夹路径对笔记进行分组
const groups = pages.groupBy(p => p.file.folder);

// 对分组后的文件夹按名称排序
groups.sort(g => g.key, 'asc').forEach(group => {
    // 为每个文件夹创建一个三级标题
    dv.header(3, group.key === currentFolder ? "本级目录文件" : group.key.replace(currentFolder + '/', ''));
    
    // 创建表格
    dv.table(
        Object.values(columns), // 表头
        group.rows
            .sort(p => p.file.name, 'asc') // 按文件名对每个分组内部的笔记进行排序
            .map(p => 
                Object.keys(columns).map(col => {
                    // 使用 eval 来解析 'file.link' 这样的深层属性
                    // 对于普通属性，直接从页面 p 中获取
                    // 如果属性不存在，则显示为空字符串
                    try {
                        return eval(`p.${col}`) || "";
                    } catch (e) {
                        return p[col] || "";
                    }
                })
            )
    );
});
```

