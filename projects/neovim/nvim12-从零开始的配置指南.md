# 从零开始的 Neovim 配置指南 —— 以 nvim12 为例

> 本文基于我自己的 Neovim 配置 [nvim12](https://github.com/luv/nvim12)，循序渐近地讲解：为什么选择 Neovim、目录结构怎么组织、每个配置项的原理是什么、以及怎么从新手成长到能自己写出这样的配置。

---

## 目录

1. [新手第一步：理解配置文件](#1-新手第一步理解配置文件)
2. [nvim12 的整体架构](#2-nvim12-的整体架构)
3. [核心配置详解](#3-核心配置详解)
   - [编辑器基础选项](#31-编辑器基础选项-options)
   - [快捷键系统](#32-快捷键系统-keymaps)
   - [插件管理：为什么用 vim.pack](#33-插件管理为什么用-vimpack)
   - [语法高亮：Tree-sitter](#34-语法高亮tree-sitter)
   - [LSP：代码智能补全与导航](#35-lsp代码智能补全与导航)
   - [格式化：conform.nvim](#36-格式化conformnvim)
   - [模糊搜索：fzf-lua](#37-模糊搜索fzf-lua)
   - [文件浏览：oil.nvim](#38-文件浏览oilnvim)
   - [迷你插件生态：mini.nvim](#39-迷你插件生态mininvim)
   - [终端集成](#310-终端集成)
   - [主题与视觉](#311-主题与视觉)
4. [配置原理总结](#4-配置原理总结)

---

## 1. 新手第一步：理解配置文件

Neovim 的配置文件放在 `~/.config/nvim/` 下。进入点只有**一个文件**：

```
~/.config/nvim/
├── init.lua          ← Neovim 启动时第一个执行的文件
├── lua/              ← 你的所有 Lua 模块
│   ├── config/       ← 配置层（选项、快捷键）
│   └── plugins/      ← 插件层（每个插件一个文件）
└── nvim-pack-lock.json  ← 插件版本锁定文件
```

**`init.lua` 就是入口。** Neovim 启动 → 执行 `init.lua` → 加载 `lua/` 下的模块。

> **原理**：Lua 的 `require("config.options")` 会去 `lua/config/options.lua` 找。点号 `.` 对应目录分隔符 `/`。这是 Lua 的模块系统，不是 Neovim 特有的。

### NVIM_APPNAME：多套配置共存

nvim12 的亮点之一是使用 `NVIM_APPNAME` 环境变量。这意味着可以在同一台机器上维护多套相互隔离的配置：

```bash
# 默认配置
nvim

# 使用 nvim12 这套配置
NVIM_APPNAME=nvim12 nvim
```

原理：Neovim 查找配置时，默认去 `~/.config/nvim/`。设置 `NVIM_APPNAME=nvim12` 后，它会改为去 `~/.config/nvim12/`，数据目录也跟着变成 `~/.local/share/nvim12/`。这让你可以大胆实验新配置而不破坏日常环境。

---

## 2. nvim12 的整体架构

打开 `~/.config/nvim12/init.lua`，只有 15 行：

```lua
require("vim._core.ui2").enable({})

require("config.keymaps")
require("config.options")
require("plugins")

vim.opt.clipboard = "unnamedplus"
vim.opt.termguicolors = true
```

它的加载顺序是有讲究的：

```
1. vim._core.ui2.enable()     ← 先启用新版 UI（0.12+ 特性）
2. config.keymaps              ← 定义快捷键（必须在插件前，否则插件的 keymap 会覆盖）
3. config.options              ← 设置基础选项
4. require("plugins")          ← 加载所有插件
5. clipboard + termguicolors   ← 最后设置的环境相关选项
```

这个顺序遵循一个原则：**先建地基，再盖房子。**

---

## 4. 核心配置详解

### 4.1 编辑器基础选项（options）

文件：`lua/config/options.lua`

```lua
-- Tab：统一使用 2 空格缩进
vim.opt.tabstop = 2        -- 一个 Tab 显示为 2 个空格的宽度
vim.opt.softtabstop = 2    -- 编辑时 Tab 产生 2 个空格
vim.opt.shiftwidth = 0     -- 智能缩进：设为 0 继承 tabstop 的值
vim.opt.expandtab = true   -- 把 Tab 键转换成空格（Python 友好）
```

> **原理**：`shiftwidth = 0` 是 Neovim 0.9+ 的新特性。设为 0 时，`>>`、`<<` 等缩进操作会自动使用 `tabstop` 的值。这样改缩进宽度时只需要改 `tabstop` 一处。

```lua
-- UI 配置
vim.opt.number = true          -- 显示绝对行号
vim.opt.relativenumber = true  -- 显示相对行号（当前行显示绝对行号，其他行显示相对距离）
vim.opt.cursorline = true      -- 高亮当前行
vim.opt.colorcolumn = "80"     -- 80 列竖线提醒
vim.opt.scrolloff = 8          -- 光标离屏幕边缘至少保留 8 行
vim.opt.showmode = false       -- 不显示 "-- INSERT --"（因为有 statusline 了）
vim.opt.cmdheight = 2          -- 命令行高度 2 行，可以看到更多历史消息
```

> **原理**：`relativenumber` + `number` 同时使用，当前行显示绝对行号，其他行显示与当前行的距离。这样执行 `10j` 向下跳 10 行时，可以直接看左边的数字，不需要心算。

```lua
-- 搜索
vim.opt.incsearch = true   -- 增量搜索：输入每个字符时即时高亮
vim.opt.hlsearch = false   -- 搜索结束后不高亮所有匹配
vim.opt.ignorecase = true  -- 默认忽略大小写
vim.opt.smartcase = true   -- 但如果搜索词包含大写字母，则区分大小写
```

> **原理**：`smartcase` 是一个很聪明的设计。搜索 `foo` 时匹配 `Foo`、`FOO`；搜索 `Foo` 时只匹配 `Foo`。它和 `ignorecase=true` 配合使用才有意义。

### 4.2 快捷键系统（keymaps）

文件：`lua/config/keymaps.lua`

nvim12 的快捷键设计遵循几个原则：

#### 原则 1：`jk` 代替 Escape

```lua
vim.keymap.set({ "i", "c" }, "jk", "<Esc>", { desc = "Escape insert mode" })
```

在 Insert 和 Command 模式下，连续按 `jk` 回到 Normal 模式。这比伸手去按左上角的 Esc 快得多，而且 `jk` 在英文中几乎不会同时出现（除了 "Djikstra" 这种罕见情况）。

#### 原则 2：空格键作为 Leader

```lua
vim.g.mapleader = " "
```

把所有自定义快捷键挂在 `<Space>` 下面。好处：空格键最大、最好按、不冲突。

#### 原则 3：分组助记

```
<Leader>f* → 文件相关（files, grep, buffers）
<Leader>t* → 终端相关（terminal, tab）
<Leader>c* → 代码相关（code action, format, diagnostics）
<Leader>r* → 重构相关（rename, restart）
```

配合 `mini.clue` 插件，按下 `<Leader>` 后会弹出一个浮动窗口列出所有可用的快捷键，不需要死记。

#### 终端快捷键：双 Esc 退出 Terminal 模式

```lua
local esc_timer = nil
vim.keymap.set("t", "<Esc>", function()
  if esc_timer then
    esc_timer:close()
    esc_timer = nil
    vim.cmd("stopinsert")
  else
    esc_timer = vim.uv.new_timer()
    esc_timer:start(300, 0, function()
      esc_timer:close()
      esc_timer = nil
    end)
  end
end, { desc = "double Esc → exit terminal mode" })
```

> **原理**：在 Terminal 模式下，按一次 `<Esc>` 本来就会传给终端内的程序（比如退出 vim 里的 insert 模式）。Neovim 默认用 `<C-\><C-n>` 退出 Terminal 模式，但双 Esc 显然更好记。这里用 `vim.uv.new_timer()` 实现了一个 300ms 内连按两次 Esc 退出 Terminal 模式的逻辑。`vim.uv` 是 Neovim 对 libuv 的封装，相当于 Node.js 里的定时器。

### 4.3 插件管理：为什么用 `vim.pack`

文件：`lua/plugins/init.lua`

nvim12 最有趣的设计决策是**不依赖任何第三方包管理器**（lazy.nvim、packer.nvim 等），直接用 Neovim 0.12 内置的 `vim.pack.add()`：

```lua
-- 自动加载 plugins/ 目录下所有 .lua 文件
for _, file in ipairs(vim.fn.readdir(vim.fn.stdpath("config") .. "/lua/plugins")) do
  if file:match("%.lua$") and file ~= "init.lua" then
    local modname = file:gsub("%.lua$", "")
    local ok, err = pcall(require, "plugins." .. modname)
    if not ok then
      vim.notify("plugins." .. modname .. " failed: " .. err, vim.log.levels.ERROR)
    end
  end
end
```

每个插件一个文件，放在 `lua/plugins/` 下，自动发现、自动加载。`pcall` 保证一个插件报错不会拖垮整个启动。

> **原理**：`vim.pack.add()` 是 Neovim 0.12 引入的内置包管理器。它会在 `~/.local/share/nvim12/site/pack/vim.pack/` 下管理插件。相比 lazy.nvim，`vim.pack` 的卖点是：零依赖、零配置、开箱即用。但代价是缺少 lazy loading 等高级功能。nvim12 的策略是：接受这个代价，因为 2026 年的硬件条件下，Neovim 本身已经足够快。

一个典型的插件文件长这样：

```lua
-- lua/plugins/fzf-lua.lua
vim.pack.add({
  { src = "https://github.com/ibhagwan/fzf-lua", name = "fzf-lua" },
})

require("fzf-lua").setup({ "fzf-native" })

vim.keymap.set("n", "<leader>ff", "<Cmd>FzfLua files<CR>")
vim.keymap.set("n", "<leader>fg", "<Cmd>FzfLua live_grep<CR>")
vim.keymap.set("n", "<leader>fb", "<Cmd>FzfLua buffers<CR>")
```

一个文件 = 声明依赖 + 配置 + 快捷键。自成一体，不互相污染。

### 4.4 语法高亮：Tree-sitter

文件：`lua/plugins/nvim-treesitter.lua`

```lua
vim.pack.add({
  { src = "https://github.com/nvim-treesitter/nvim-treesitter", name = "nvim-treesitter" },
})

vim.api.nvim_create_autocmd("FileType", {
  pattern = "*",
  callback = function(ev)
    local lang = vim.treesitter.language.get_lang(vim.bo[ev.buf].filetype)
    if lang and vim.treesitter.language.add(lang) then
      vim.treesitter.start(ev.buf, lang)
    end
  end,
})
```

> **原理**：传统的 Vim 语法高亮用正则表达式，Tree-sitter 用 CST（具体语法树）。区别就像「用正则解析 HTML」和「用浏览器引擎解析 HTML」——前者在复杂的嵌套结构面前必然失败，后者是真正的解析。

`vim.treesitter.language.add(lang)` 会下载对应语言的 parser。`vim.treesitter.start()` 启动增量解析。解析后的语法树不仅用于高亮，还能用于代码折叠、缩进、文本对象选择（如 `vii` 选中块内部）。

nvim12 的做法是**自动检测文件类型并启动 Tree-sitter**，不需要手动声明支持哪些语言。

### 4.5 LSP：代码智能补全与导航

文件：`lua/plugins/nvim-lspconfig.lua`

```lua
vim.pack.add({
  { src = "https://github.com/neovim/nvim-lspconfig", name = "nvim-lspconfig" },
})

vim.lsp.enable("pyright")   -- Python
vim.lsp.enable("lua_ls")    -- Lua
vim.lsp.enable("clangd")    -- C/C++

vim.api.nvim_create_autocmd("LspAttach", {
  callback = function(ev)
    local map = function(keys, func, desc)
      vim.keymap.set("n", keys, func, { buffer = ev.buf, desc = desc })
    end

    map("gd", vim.lsp.buf.definition, "Goto definition")
    map("gr", vim.lsp.buf.references, "Goto references")
    map("K", vim.lsp.buf.hover, "Hover documentation")
    map("<leader>rn", vim.lsp.buf.rename, "Rename symbol")
    map("<leader>ca", vim.lsp.buf.code_action, "Code action")
    map("[d", vim.diagnostic.goto_prev, "Prev diagnostic")
    map("]d", vim.diagnostic.goto_next, "Next diagnostic")
    -- ... 更多
  end,
})
```

> **原理**：LSP（Language Server Protocol）是微软定义的一个协议。编辑器和语言服务器通过 JSON-RPC 通信。编辑器发送请求（比如"第 10 行第 5 列的变量是什么？"），服务器返回结果（定义位置、类型信息、诊断错误等）。Neovim 0.12 的 `vim.lsp.enable()` 是最高层级的 API，一行代码启动一个语言服务器。

文件：`lua/plugins/mason.lua`

```lua
vim.pack.add({
  { src = "https://github.com/mason-org/mason.nvim", name = "mason" },
})

require('mason').setup()
```

> **原理**：mason.nvim 是语言服务器的包管理器。它帮你下载 pyright、clangd、lua_ls 这些服务器二进制文件，放到统一的目录里，自动加入 PATH。不需要手动 `npm install -g pyright` 或 `pip install`。

#### 补全：`mini.completion`

nvim12 没有用 nvim-cmp，而是用了 `mini.completion`——它是 mini.nvim 生态中的一个轻量级补全引擎：

```lua
require("mini.completion").setup()
```

> **原理**：和 nvim-cmp 不同，`mini.completion` 不需要配置 source，它会自动从 LSP client 获取补全项。它的设计哲学是「默认即合理」，牺牲一些可定制性换取零配置体验。

### 4.6 格式化：conform.nvim

文件：`lua/plugins/conform.lua`

```lua
conform.setup({
  formatters_by_ft = {
    lua = { "stylua" },
    python = { "isort", "black" },       -- 先 isort 排序导入，再 black 格式化
  },
  format_on_save = {
    timeout_ms = 500,
    lsp_format = "fallback",             -- 如果 formatter 不支持，回退到 LSP 格式化
  },
})

vim.keymap.set({ "n", "x" }, "<leader>cf", function()
  conform.format({ lsp_format = "fallback" })
end)
```

> **原理**：`format_on_save` 在保存文件时自动运行格式化。`lsp_format = "fallback"` 的意思是优先用 conform 配置的 formatter（stylua/black），如果文件类型没配置 formatter，就用 LSP 提供的格式化能力。`timeout_ms = 500` 防止格式化卡住太久。

Python 的 `{ "isort", "black" }` 是**顺序执行**：先 isort 整理 import 顺序，再 black 格式化代码风格。如果改成 `{ "prettierd", "prettier", stop_after_first = true }`，就是找到第一个可用的就停。

### 4.7 模糊搜索：fzf-lua

文件：`lua/plugins/fzf-lua.lua`

```lua
vim.pack.add({
  { src = "https://github.com/ibhagwan/fzf-lua", name = "fzf-lua" },
})

require("fzf-lua").setup({ "fzf-native" })
```

> **原理**：`fzf-native` 是一个 C 扩展，用 Rust 写的模糊匹配算法（skim 算法）。比纯 Lua 的模糊匹配快 10-100 倍。在大项目里搜文件或 grep 时，这个差别非常明显。

快捷键：
- `<Leader>ff` — 模糊搜索文件名
- `<Leader>fg` — 项目内模糊 grep
- `<Leader>fb` — 模糊搜索已打开的 buffer

### 4.8 文件浏览：oil.nvim

文件：`lua/plugins/oil.lua`

```lua
require("oil").setup()
vim.keymap.set("n", "<Leader>e", "<Cmd>Oil<CR>")
```

> **原理**：oil.nvim 和传统文件树（NvimTree、nerdtree）的根本区别：它把目录当作一个可编辑的文本 buffer。你可以像编辑文本一样重命名文件、创建目录、移动文件。所有操作用 Vim 的文本编辑原语完成，不需要学习新快捷键。这是一种「少即是多」的设计。

### 4.9 迷你插件生态：mini.nvim

文件：`lua/plugins/mini.lua`

nvim12 用了 mini.nvim 生态中的 19 个模块，它们一起提供了一套完整的、风格统一的体验：

| 模块 | 功能 | 原理 |
|------|------|------|
| `mini.icons` | 文件图标 | 用 Nerd Font 字符显示文件类型图标 |
| `mini.surround` | 括号/引号操作 | `sa` 添加环绕，`sd` 删除环绕 |
| `mini.pairs` | 自动配对 | 输入 `(` 自动补 `)`，并有智能删除 |
| `mini.ai` | 文本对象增强 | `va)` 选括号内容 + 括号本身 |
| `mini.jump2d` | 二维跳转 | 类似 easymotion，按两个字符跳到屏幕任意位置 |
| `mini.snippets` | 代码片段 | 轻量 snippet 引擎，用 Lua 定义 |
| `mini.completion` | 补全 | 从 LSP 获取补全，零配置 |
| `mini.statusline` | 状态栏 | 显示模式、文件名、LSP 状态等 |
| `mini.cursorword` | 光标词高亮 | 高亮当前光标下的其他相同单词 |
| `mini.diff` | diff 增强 | 更直观的差异显示 |
| `mini.cmdline` | 命令行增强 | 显示命令历史和建议 |
| `mini.pick` | 选择器 | 文件、buffer、help 等选择界面 |
| `mini.indentscope` | 缩进指示线 | 垂直缩进线，字符设为 `┃` |
| `mini.hipatterns` | 模式高亮 | 高亮 FIXME/TODO 注释和颜色代码 |
| `mini.starter` | 启动页 | 自定义 ASCII logo 的启动屏幕 |
| `mini.clue` | 快捷键提示 | 按 `<Leader>` 弹出快捷键菜单 |
| `mini.sessions` | 会话管理 | 保存/恢复窗口布局 |
| `mini.keymap` | 快捷键增强 | 多步按键映射（如 Tab 的行为） |
| `mini.clue` | 快捷键提示 | 按 Leader 弹出所有快捷键菜单 |

**mini.nvim 的哲学**：每个模块是独立的 Lua 文件，没有共享内部状态，可以单独使用。所有模块保持一致的 API 风格（`require("mini.xxx").setup()`）。这避免了插件碎片化——你不需要记忆 19 个不同插件、不同文档、不同配置方式。

#### 重点：mini.clue

```lua
local miniclue = require("mini.clue")
miniclue.setup({
  triggers = {
    { mode = { "n", "x" }, keys = "<Leader>" },  -- 按 Leader 弹出提示
    { mode = "n", keys = "[" },                    -- 按 [ 弹出提示
    { mode = "n", keys = "]" },                    -- 按 ] 弹出提示
    { mode = { "n", "x" }, keys = "g" },           -- 按 g 弹出提示
    -- ...
  },
  window = {
    config = { width = "45" },
    delay = 200,                                   -- 200ms 后弹出
  },
})
```

> **原理**：当你按下 `<Leader>` 并停顿 200ms，`mini.clue` 会弹出一个浮动窗口，列出所有以 `<Leader>` 开头定义的快捷键。这意味着你不需要背快捷键——肌肉记忆建立之前，有提示可看。

#### 重点：mini.hipatterns

```lua
require("mini.hipatterns").setup({
  highlighters = {
    fixme = { pattern = "%f[%w]()FIXME()%f[%W]", group = "MiniHipatternsFixme" },
    todo  = { pattern = "%f[%w]()TODO()%f[%W]",  group = "MiniHipatternsTodo" },
    hex_color = require("mini.hipatterns").gen_highlighter.hex_color(),
  },
})
```

> **原理**：`mini.hipatterns` 用 Lua 正则实时高亮 buffer 中的特定模式。`%f[%w]` 是 Lua 的边界匹配（word-boundary），确保 `FIXME` 只匹配完整的单词，不会匹配 `FIXME_something`。`hex_color` 会把 `#ff0000` 等颜色代码真正染成相应的颜色。

### 4.10 终端集成

文件：`lua/config/keymaps.lua`

```lua
vim.keymap.set({ "n", "t" }, "<C-_>", function()
  local buf = find_term_buf()
  if buf then
    local win = find_term_win(buf)
    if win then
      vim.api.nvim_win_close(win, true)           -- 有可见的终端 → 关闭
    else
      vim.cmd("botright 12split | buffer " .. buf) -- 有隐藏的终端 → 显示
    end
  else
    vim.cmd("botright 12split | terminal")         -- 没有终端 → 创建
  end
end, { silent = true, desc = "toggle bottom terminal" })
```

> **原理**：这个函数实现了三态切换：
> 1. **无终端** → 在底部创建一个 12 行高的终端
> 2. **终端存在但不可见** → 在底部重新打开
> 3. **终端可见** → 关闭窗口（进程不杀，buffer 保留）

```
vim.api.nvim_list_bufs()  → 所有 buffer 列表
vim.bo[buf].buftype       → buffer 类型（"terminal" = 终端）
vim.api.nvim_list_wins()  → 所有窗口列表
vim.api.nvim_win_get_buf() → 窗口对应的 buffer
```

`<Leader>tt` 的版本类似，但是用 `tabnew` 代替 split，实现**全屏终端**的切换。

#### Tab 切换

```lua
vim.keymap.set("n", "<C-h>", "gT")  -- 上一个 tab
vim.keymap.set("n", "<C-l>", "gt")  -- 下一个 tab
```

`Ctrl+h` / `Ctrl+l` 的肌肉记忆来自 Vim 的分屏导航，这里映射到 tab 切换更符合直觉。

### 4.11 主题与视觉

文件：`lua/plugins/colorscheme.lua`

```lua
vim.pack.add({
  { src = "https://github.com/catppuccin/nvim",    name = "catppuccin" },
  { src = "https://github.com/rose-pine/neovim",   name = "rose-pine" },
  { src = "https://github.com/folke/tokyonight.nvim", name = "tokyonight" },
  { src = "https://github.com/ellisonleao/gruvbox.nvim", name = "gruvbox.nvim" },
})

vim.cmd.colorscheme("tokyonight-moon")   -- 当前激活
-- vim.cmd.colorscheme("catppuccin-macchiato")     -- 备选
-- vim.cmd.colorscheme("rose-pine-moon")            -- 备选
-- vim.cmd.colorscheme("gruvbox")                   -- 备选
```

四个主题已安装，一句话切换。注释不是删掉的代码，而是**热备选项**——想换主题时取消一行、注释另一行即可。

```lua
-- init.lua
vim.opt.termguicolors = true
```

> **原理**：`termguicolors` 启用 24-bit 真彩色。大多数现代终端（Kitty、Alacritty、WezTerm、iTerm2）都支持。如果不开启，colorscheme 只能用 256 色，显示效果大打折扣。

文件：`lua/plugins/smear-cursor.lua`

```lua
require("smear_cursor").setup({})
```

这个插件让光标移动带有平滑的拖尾动画。纯视觉效果，但能让编辑体验提升一个档次。

---

## 5. 配置原理总结

nvm12 这套配置反映了几条核心理念：

### 理念 1：极简主义

- 没有 lazy.nvim，用 `vim.pack`，减少一层抽象
- 不用 nvim-cmp，用 `mini.completion`，少写 200 行 cmp 配置
- 不用 NvimTree，用 oil.nvim，用已有技能（文本编辑）做文件管理
- 每个文件短小精悍（最长 126 行），而不是放在一个 2000 行的 init.lua 里

### 理念 2：内聚优于解耦

每个插件文件包含三件事：声明依赖 + 配置 + 绑快捷键。删除一个插件 = 删除一个文件，不留痕迹。

### 理念 3：键盘优先

- `jk` 替代 Esc——手不离开主键盘区
- 双 Esc 退出终端——减少心理负担
- Leader 键分层——避免快捷键冲突
- `mini.clue` 做记忆辅助——不用死记硬背

### 理念 4：默认即合理

- Treesitter 自动开启，不手动声明语言列表
- Mason 自动管理 LSP 服务器
- conform 保存时自动格式化
- `mini.completion` 零配置补全

### 理念 5：渐进式配置

- 四个主题已安装但只有一个激活
- LSP 只启用了三个（Python、Lua、C/C++），按需添加
- 插件按文件隔离，方便渐进替换

---

## 给新手的建议

如果你从零开始，推荐的顺序是：

1. **先装好 Neovim 0.12+，复制这套配置跑起来**
2. **练习基本操作**：`hjkl` 移动、`jk` 退出、`<Leader>ff` 打开文件
3. **逐步理解**：打开 `lua/config/options.lua`，把每个选项的注释读一遍；不明白的去 `:help 'option_name'`
4. **按需调整**：你不写 Python？删掉 conform 里的 python 配置。不用 C++？删掉 clangd
5. **添加新语言**：需要 Rust？加 `vim.lsp.enable("rust_analyzer")` + conform 里加 `rust = { "rustfmt" }`

---

> 配置仓库地址：[github.com/luv/nvim12](https://github.com/luv/nvim12)
>
> Neovim 的哲学是「可编程的编辑器」。理解了配置的原理，你的编辑器就不再是黑盒，而是被你完全掌控的工具。
