# Lua 在 Neovim 中的使用指南

## 一、Lua 语言简介

Lua 是一个轻量级脚本语言，设计目标是简洁、可嵌入。语法接近 Python/Ruby，但更小更省资源。Neovim 内嵌了 LuaJIT（Lua 的 JIT 编译器版本），所以 Lua 代码在 Neovim 里执行非常快。

几个最基本的 Lua 语法（后面会遇到）：

```lua
-- 注释用 --
local x = 1          -- 变量（不加 local 会变成全局变量，永远用 local）
local t = {1, 2, 3} -- 表（table），Lua 唯一的数据结构，兼做数组和字典
t.key = "value"      -- 字典写法
print(t[1])          -- 数组从 1 开始索引（不是 0！）
```

---

## 二、为什么 Neovim 要用 Lua

传统 Vim 用 Vimscript 配置，但 Vimscript 语法怪异、执行慢、缺少很多现代语言特性。Neovim 在 Vimscript 之外完整支持 Lua，用 Lua 配置的优势：

- **语法更现代**：变量、循环、条件、函数都是标准写法
- **执行更快**：LuaJIT 即时编译，比 Vimscript 快数倍
- **更丰富的 API**：Neovim 提供了完整的 Lua 接口来操控编辑器
- **可以直接写内联函数**：映射、回调直接写 `function() ... end`，不必定义单独的脚本函数

---

## 三、Neovim 的 Lua API 三层结构

Neovim 把 API 分了三层，你可以混用：

| 层级 | 访问方式 | 来源 |
|------|---------|------|
| **Vim API**（旧 Vim 命令/函数） | `vim.cmd()` 执行 Ex 命令<br>`vim.fn` 调用 Vimscript 函数 | 继承自 Vim |
| **Nvim API**（C 写的底层 API） | `vim.api` | Neovim 自己用 C 实现的 |
| **Lua stdlib**（Lua 原生包装） | `vim.*` 其它内容（如 `vim.keymap`、`vim.opt` 等） | 专门为 Lua 设计的便利层 |

简单说：能用 `vim.keymap.set()` 就不要用 `vim.cmd("noremap ...")`。Lua stdlib 是最方便的选择。

---

## 四、如何运行 Lua 代码

### 1. 在 Neovim 命令行直接执行

```
:lua print("Hello!")
:lua =package       -- 打印变量的值（等同于 :lua vim.print(...)）
```

**注意**：每个 `:lua` 是一个独立的局部作用域。所以这样不行：

```vim
:lua local foo = 1
:lua print(foo)     " 输出 nil，不是 1！
```

### 2. 执行外部 Lua 文件

```vim
:source ~/myconfig.lua
```

### 3. 在 Vimscript 文件中嵌入 Lua

```vim
lua << EOF
  local tbl = {1, 2, 3}
  for k, v in ipairs(tbl) do
    print(v)
  end
EOF
```

---

## 五、配置文件：init.lua

Neovim 支持 `init.vim` 或 `init.lua` 作为配置文件（二选一，不能同时存在）。放在你的配置目录里（运行 `:echo stdpath('config')` 可以看到路径，默认是 `~/.config/nvim/`）。

典型的 `init.lua` 结构：

```lua
-- 基础设置
vim.opt.number = true
vim.opt.expandtab = true

-- 按键映射
vim.keymap.set('n', '<Leader>w', '<cmd>w<cr>')

-- 加载插件管理器
require('lazy').setup({...})
```

**自动加载脚本**：放在 `plugin/` 目录的 Lua 文件会在启动时自动执行（类似于 Vimscript 的 `plugin/`）。

---

## 六、Lua 模块系统：`require()`

这是 Neovim 配置中最核心的概念之一。`require()` 用来加载 Lua 模块。

**目录结构要求**：模块文件必须放在 `lua/` 目录下（你的配置目录或任何 `runtimepath` 路径的 `lua/` 子目录）。

```
~/.config/nvim/
├── init.lua
├── lua/
│   ├── mymodule.lua        -- require("mymodule")
│   └── other/
│       ├── init.lua        -- require("other")  加载这个
│       └── util.lua        -- require("other.util")
└── plugin/
```

**`require()` 的关键行为**：

- 自动搜索 `runtimepath` 下所有 `lua/` 目录
- **只执行一次**，之后返回缓存的结果（第二次调用不会重新读取磁盘）
- 如果要重新加载，需要手动清除缓存：

```lua
package.loaded['mymodule'] = nil
require('mymodule')
```

**错误处理**：可以用 `pcall` 安全加载：

```lua
local ok, mymod = pcall(require, 'unstable_plugin')
if not ok then
  print("加载失败")
else
  mymod.setup()
end
```

这就是 LazyVim、Lazy.nvim 等插件管理器延迟加载的底层机制——在真正需要的时候才 `require`。

---

## 七、从 Lua 调用 Vim 的东西

### 调用 Vim 命令：`vim.cmd()`

```lua
vim.cmd("colorscheme habamax")
vim.cmd("set number")

-- 多行命令（字面字符串）
vim.cmd([[
  highlight Error guibg=red
  highlight link Warning Error
]])

-- 面向对象的调用方式
vim.cmd.colorscheme("habamax")
vim.cmd.highlight({ "Error", "guibg=red" })
```

### 调用 Vimscript 函数：`vim.fn`

```lua
print(vim.fn.printf('Hello from %s', 'Lua'))
local reversed = vim.fn.reverse({'a', 'b', 'c'})  -- 自动类型转换
print(vim.fn.expand('%'))      -- 当前文件名
print(vim.fn.has('nvim-0.10')) -- 检查功能
```

---

## 八、变量的读写：`vim.g` / `vim.b` / `vim.w` / `vim.t` / `vim.env`

对应 Vimscript 的各作用域变量：

| Lua 写法 | Vimscript 等价 | 含义 |
|----------|---------------|------|
| `vim.g.myvar` | `g:myvar` | 全局变量 |
| `vim.b.myvar` | `b:myvar` | 当前 buffer 的变量 |
| `vim.w.myvar` | `w:myvar` | 当前 window 的变量 |
| `vim.t.myvar` | `t:myvar` | 当前 tab 的变量 |
| `vim.v.myvar` | `v:myvar` | Vim 预定义变量 |
| `vim.env.PATH` | `$PATH` | 环境变量 |

```lua
vim.g.my_global = { key = "value", num = 300 }

-- 指定编号
vim.b[2].myvar = 1       -- buffer 2
vim.w[1005].myvar = true -- window ID 1005

-- 删除变量
vim.g.myvar = nil
```

**重要陷阱**：你不能直接修改 table 型变量的子字段：

```lua
vim.g.my_global.key = 400   -- 不会生效！
-- 正确做法：
local tmp = vim.g.my_global
tmp.key = 400
vim.g.my_global = tmp
```

---

## 九、选项的设置：`vim.opt` 和 `vim.o`

有两套互补的方式，选一个用即可。

### `vim.opt`（推荐，处理列表/集合最方便）

```lua
vim.opt.smarttab = true       -- :set smarttab
vim.opt.wildignore = { '*.o', '*.a', '__pycache__' }  -- 列表选项
vim.opt.listchars = { space = '_', tab = '>~' }        -- 键值对选项
vim.opt.formatoptions = { n = true, j = true }          -- 标志集合选项

-- 追加/前置/移除
vim.opt.shortmess:append({ I = true })
vim.opt.wildignore:prepend('*.o')
vim.opt.whichwrap:remove({ 'b', 's' })

-- 读取值必须用 :get()
print(vim.opt.smarttab:get())  --> false
```

### `vim.o` / `vim.go` / `vim.bo` / `vim.wo`（更直接，模拟 Vimscript 的 `&option`）

```lua
vim.o.smarttab = false       -- 全局选项
vim.go.shiftwidth = 4        -- 全局默认值
vim.bo.shiftwidth = 4        -- buffer 局部选项
vim.wo.number = true         -- window 局部选项

-- 指定 buffer/window
vim.bo[4].expandtab = true   -- buffer 4
vim.wo[1005].number = true   -- window ID 1005

-- 可以直接读取
print(vim.o.smarttab)  --> false
```

**选哪个？** 列表型和集合型选项用 `vim.opt` 更方便；简单的单值选项用 `vim.o` 更直接。可以混用，比如 `vim.bo.shiftwidth = 4`。

---

## 十、按键映射：`vim.keymap.set()`

这是配置中最常用的部分。

```lua
-- 基本格式：vim.keymap.set(模式, 按键, 动作, [选项])

-- 执行 Vim 命令
vim.keymap.set('n', '<Leader>w', '<cmd>write<cr>')

-- 执行 Lua 函数
vim.keymap.set('n', '<Leader>h', function() print("Hello") end)

-- 多模式
vim.keymap.set({'n', 'v'}, '<Leader>x', '<cmd>bd<cr>')

-- 调用模块函数
vim.keymap.set('n', '<Leader>f', require('telescope.builtin').find_files)
```

**常用选项**（第四个参数是一个 table）：

| 选项 | 含义 |
|------|------|
| `desc = "描述"` | 给映射加描述（插件强烈建议加） |
| `silent = true` | 不显示执行信息 |
| `buffer = 0` | 只在当前 buffer 生效 |
| `expr = true` | rhs 是表达式，用返回值当输入 |
| `remap = true` | 允许递归映射（默认是非递归的 `noremap`） |

```lua
vim.keymap.set('n', '<Leader>pl', require('plugin').action,
  { silent = true, desc = 'Plugin action' })
```

**删除映射**：

```lua
vim.keymap.del('n', '<Leader>ex1')
```

---

## 十一、自动命令（Autocommands）：事件触发

当特定事件发生时（打开文件、保存、切换窗口等）自动执行代码。

### 创建自动命令

```lua
vim.api.nvim_create_autocmd({"BufEnter", "BufWinEnter"}, {
  pattern = {"*.c", "*.h"},
  command = "echo '进入了 C 文件'",   -- Vim 命令
  -- 或者用 callback 代替 command:
  -- callback = function() print("进入了 C 文件") end,
})
```

**回调函数会收到一个事件参数 `ev`**，常用字段：

| 字段 | 含义 |
|------|------|
| `ev.match` | 匹配到的模式（如文件名） |
| `ev.buf` | 触发事件的 buffer 号 |
| `ev.file` | 触发事件的文件名 |

```lua
-- 利用 ev.buf 给特定文件类型设置 buffer 局部映射
vim.api.nvim_create_autocmd("FileType", {
  pattern = "lua",
  callback = function(ev)
    vim.keymap.set('n', 'K', vim.lsp.buf.hover, { buffer = ev.buf })
  end
})
```

> **注意**：如果回调函数自己需要参数，要包一层 `function() ... end`：
>
> ```lua
> vim.api.nvim_create_autocmd('TextYankPost', {
>   callback = function() vim.hl.on_yank() end
> })
> ```

**Buffer 局部自动命令**：

```lua
vim.api.nvim_create_autocmd("CursorHold", {
  buffer = 0,   -- 0 代表当前 buffer
  callback = function() print("hold") end,
})
```

### 分组：防止重复加载

自动命令放在 `init.lua` 里有个经典问题：每次重新 source 配置都会再注册一遍。解决方法是用 **augroup**：

```lua
-- Vimscript 的经典写法等价于：
local mygroup = vim.api.nvim_create_augroup('myconfig', { clear = true })

vim.api.nvim_create_autocmd({ 'BufNewFile', 'BufRead' }, {
  pattern = '*.html',
  group = mygroup,
  command = 'set shiftwidth=4',
})
vim.api.nvim_create_autocmd({ 'BufNewFile', 'BufRead' }, {
  pattern = '*.html',
  group = 'myconfig',  -- 也可以用名字
  command = 'set expandtab',
})
```

`{ clear = true }` 的意思是：如果这个组已经存在，先把组里之前的自动命令全部删掉，再创建新的。这样无论 source 多少次都不会重复。

### 删除自动命令

```lua
-- 删除所有 BufEnter 事件的自动命令
vim.api.nvim_clear_autocmds({event = "BufEnter"})

-- 删除某个组的
vim.api.nvim_clear_autocmds({group = "myconfig"})
```

---

## 十二、用户命令：自定义 Vim 命令

创建你自己的 `:MyCommand` 命令：

```lua
vim.api.nvim_create_user_command('Test', 'echo "It works!"', {})
vim.cmd.Test()   -- 执行它

-- 带参数的 Lua 函数版本
vim.api.nvim_create_user_command('Upper',
  function(opts)
    print(string.upper(opts.fargs[1]))
  end,
  { nargs = 1, desc = "转大写" }
)
vim.cmd.Upper('hello')  -- 输出 HELLO
```

**回调参数 `opts`** 包含：

| 字段 | 含义 |
|------|------|
| `opts.fargs` | 按空格分割的参数列表 |
| `opts.bang` | 是否有 `!` |
| `opts.line1`/`opts.line2` | 范围行号 |
| `opts.range` | 范围有几项（0/1/2） |
| `opts.count` | 前置数字 |

**删除命令**：

```lua
vim.api.nvim_del_user_command('Upper')
```

---

## 十三、新手写配置的典型流程

以改变你的 `~/.config/lazyvim/init.lua` 为例，一个典型的个人配置文件骨架：

```lua
-- 1. 基础选项
vim.opt.number = true          -- 行号
vim.opt.relativenumber = true  -- 相对行号
vim.opt.expandtab = true       -- Tab 变空格
vim.opt.shiftwidth = 2         -- 缩进 2 格
vim.opt.mouse = 'a'            -- 启用鼠标

-- 2. 按键映射
vim.keymap.set('n', '<Leader>w', '<cmd>w<cr>', { desc = '保存' })
vim.keymap.set('n', '<Leader>q', '<cmd>q<cr>', { desc = '退出' })

-- 3. 自动命令（放在一个组里防止重复）
local aug = vim.api.nvim_create_augroup('myconfig', { clear = true })
vim.api.nvim_create_autocmd('TextYankPost', {
  group = aug,
  callback = function() vim.hl.on_yank() end,
  desc = '高亮复制内容',
})

-- 4. 加载插件管理器（以 Lazy.nvim 为例，你的 LazyVim 已配好）
-- 在 LazyVim 中这部分已经配置好了，你只需要在 lua/plugins/ 下加文件
```

---

**总结**：Lua 在 Neovim 中的核心思路就是——用 `vim.xxx` API 取代 Vimscript 的 `set`/`map`/`au` 等命令。API 相当完整，基本上 Vimscript 能做的事用 Lua 都能做，而且更简洁。你的 LazyVim 配置本质上就是一个大型的 Lua 项目，所有插件配置都在 `lua/plugins/` 下以 `require` 模块的方式组织。
