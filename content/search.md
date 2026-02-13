---
title: "搜索"
layout: "search"
url: "/search/"
summary: "搜索文章"
---

<div class="search-container">
  <input type="text" id="searchInput" placeholder="🔍 搜索文章标题、内容、标签..." autofocus />
  <p class="search-hint">按 <kbd>/</kbd> 快速聚焦搜索框，按 <kbd>Esc</kbd> 退出</p>
</div>

<style>
.search-container {
  margin: 20px 0 40px 0;
}

.search-hint {
  margin-top: 10px;
  font-size: 0.875rem;
  color: var(--secondary);
  text-align: center;
}

.search-hint kbd {
  padding: 2px 6px;
  background: var(--code-bg);
  border: 1px solid var(--border);
  border-radius: 4px;
  font-family: monospace;
  font-size: 0.875rem;
}
</style>
