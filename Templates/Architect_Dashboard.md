```dataviewjs
// === 架构师控制台 (修复版) ===

dv.header(1, "🚀 任务与知识追踪");

// 1. 定义数据源：排除 "Templates" 文件夹
let pages = dv.pages('-"Templates"');

// 2. 统计各状态数量
// 注意：这里匹配的是你在 YAML 中写的 status: #Status/Inbox
let inbox = pages.filter(p => p.status == "#Status/Inbox").length;
let coding = pages.filter(p => p.status == "#Status/Coding").length;
let refactoring = pages.filter(p => p.status == "#Status/Refactoring").length;

// 渲染统计行
dv.paragraph(`**📥 待处理 (Inbox):** ${inbox} | **💻 开发中 (Coding):** ${coding} | **♻️ 待重构 (Refactoring):** ${refactoring}`);

dv.header(2, "🔥 正在攻坚 (High Complexity & Active)");

// 3. 筛选高优任务
// 逻辑：状态不是“已归档” 且 复杂度字段存在 且 长度>=3 (即3星以上)
let active_hard_tasks = pages.filter(p => 
    p.status != "#Status/Archived" && 
    p.complexity && 
    p.complexity.length >= 3
);

// 4. 渲染表格
if (active_hard_tasks.length > 0) {
    dv.table(
        ["笔记/任务", "技术栈", "复杂度", "最后修改"],
        active_hard_tasks
        .sort(p => p.file.mtime, "desc")
        .map(p => [
            p.file.link,
            p.tech_stack ? p.tech_stack : "-", // 如果没填技术栈显示 -
            p.complexity,
            p.file.mtime.toFormat("MM-dd HH:mm")
        ])
    );
} else {
    dv.paragraph("✅ 目前没有高复杂度积压任务！");
}

dv.header(2, "📂 最近更新的 UE5 模块");

// 5. 渲染列表
dv.list(pages
    .filter(p => p.tech_stack && p.tech_stack.includes("UE5")) // 模糊匹配，防止你写成 #Tech/UE5 但这里只查 "UE5"
    .sort(p => p.file.mtime, "desc")
    .limit(5)
    .file.link
);