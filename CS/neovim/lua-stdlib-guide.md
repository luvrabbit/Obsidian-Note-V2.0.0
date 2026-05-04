# Neovim Lua stdlib 全览

## 什么是 Lua stdlib

Neovim 的 Lua 标准库（stdlib）就是 `vim` 这个全局模块及其所有子模块。它**始终自动加载**，不需要 `require("vim")`。

你可以在 Neovim 里运行 `:lua vim.print(vim)` 看看它有多少东西——它是一个包含几百个函数和子模块的巨大 table。

---

## 三层结构回顾

`vim.*` 的内容按来源分三层：

| 命名空间 | 来源 | 示例 |
|---------|------|------|
| `vim.api.*` | Neovim C API | `vim.api.nvim_get_current_buf()` |
| `vim.fn.*` | Vimscript 函数 | `vim.fn.expand('%')` |
| `vim.*` 其它 | 专为 Lua 设计的便利库 | `vim.keymap.set()`, `vim.inspect()` |

stdlib 主要关注第三层——专门为 Lua 用户编写的工具函数和子模块。

---

## 一、核心工具函数（最常用）

### 调试与打印

```lua
vim.print("hello", {a=1, b=2})   -- 美化的 print，会递归展示 table
vim.inspect({a=1, b=2})          -- 返回美化的字符串而不直接输出
vim.notify("保存成功", vim.log.levels.INFO)  -- 弹通知（可被插件接管）
vim.notify_once("只提示一次")      -- 相同消息不会重复弹出
vim.deprecate("old_func", "new_func", "0.12")  -- 标记函数已废弃
```

### 类型判断

```lua
vim.isarray({1, 2, 3})     -- true（纯整数键从 1 开始的 table）
vim.islist({1, 2, 3})      -- true（连续的整数键，无空洞）
vim.is_callable(func)       -- 判断是否可调用（函数或带 __call 的 table）
```

### Table 工具

```lua
-- 合并
vim.tbl_extend("force", {a=1}, {b=2})           -- 浅合并 -> {a=1, b=2}
vim.tbl_deep_extend("force", {a={x=1}}, {a={y=2}})  -- 深合并 -> {a={x=1, y=2}}

-- 遍历
vim.tbl_map(fn, t)       -- 对每个值执行 fn，返回新 table
vim.tbl_filter(fn, t)    -- 筛选满足条件的值
vim.tbl_keys(t)          -- 返回所有键
vim.tbl_values(t)        -- 返回所有值
vim.tbl_count(t)         -- 统计非 nil 值的数量
vim.tbl_isempty(t)       -- 判断是否为空
vim.tbl_contains(t, val) -- 判断是否包含某个值
vim.tbl_get(t, "a", "b") -- 安全获取深层嵌套值：t.a.b

-- 拷贝
vim.deepcopy(orig)       -- 深拷贝，处理循环引用
vim.deep_equal(a, b)     -- 深层比较是否相等
vim.spairs(t)            -- 按键排序遍历 (sorted pairs)
```

### List 工具

```lua
vim.list_contains({1,2,3}, 2)     -- true
vim.list_extend(dst, src)         -- 把 src 追加到 dst
vim.list_slice({1,2,3,4}, 2, 3)  -- {2, 3}
vim.list.unique({1,2,2,3})       -- 原地去重
```

### 字符串工具

```lua
vim.split("a,b,c", ",")          -- {"a", "b", "c"}
vim.gsplit("a,b,c", ",")          -- 惰性迭代器版本（省内存）
vim.trim("  hello  ")             -- "hello"
vim.startswith("hello", "he")     -- true
vim.endswith("hello", "lo")       -- true
vim.pesc("foo.bar")              -- 转义 Lua pattern 中的魔法字符 -> "foo%.bar"
vim.stricmp("ABC", "abc")        -- 0（大小写不敏感比较）
```

### 编码与加密

```lua
vim.base64.encode("hello")        -- "aGVsbG8="
vim.base64.decode("aGVsbG8=")    -- "hello"
vim.iconv("你好", "utf-8", "gbk") -- 编码转换
```

### JSON / MessagePack

```lua
vim.json.encode({a=1})            -- 返回 JSON 字符串
vim.json.decode('{"a":1}')        -- {a=1}
vim.mpack.encode({1,2})           -- MessagePack 二进制
vim.mpack.decode(binary_string)
```

### 时间与调度

```lua
-- 延迟执行（一次性定时器）
vim.defer_fn(function() print("3秒后") end, 3000)

-- 推迟到主事件循环（安全调用 API）
vim.schedule(function() vim.cmd("echo 'hi'") end)

-- 把回调包一层 schedule（常用于异步回调）
local safe_fn = vim.schedule_wrap(function() vim.notify("done") end)

-- 等待直到条件成立或超时
vim.wait(5000, function() return some_condition end)
```

### 参数校验（插件作者常用）

```lua
vim.validate({
  name = { name, "string" },
  opts = { opts, "table", true },       -- true = 可选
  count = { count, "number", false },    -- false = 必填
})
-- 不符合类型会抛出友好错误信息
```

---

## 二、Vimscript 桥接层

### 变量作用域

| Lua | Vimscript | 说明 |
|-----|----------|------|
| `vim.g.foo` | `g:foo` | 全局 |
| `vim.b.foo` | `b:foo` | 当前 buffer |
| `vim.w.foo` | `w:foo` | 当前 window |
| `vim.t.foo` | `t:foo` | 当前 tab |
| `vim.v.foo` | `v:foo` | 预定义 Vim 变量 |
| `vim.env.PATH` | `$PATH` | 环境变量 |

```lua
-- 指定编号
vim.b[3].foo = "hello"    -- buffer 3
vim.w[1005].foo = true    -- window ID 1005
-- 删除
vim.g.foo = nil
```

### 选项

```lua
-- 直接风格（读值方便）
vim.o.number = true        -- :set number
vim.bo.tabstop = 4         -- :setlocal tabstop=4
vim.wo.cursorline = true   -- window 选项
vim.go.autochdir = false   -- 全局默认值

-- 对象风格（列表/集合选项更方便）
vim.opt.wildignore = {"*.o", "*.a"}
vim.opt.formatoptions:append("j")
vim.opt.listchars = {space = "_", tab = ">~"}
print(vim.opt.listchars:get())  -- 读值

-- 局部/全局版本
vim.opt_local.tabstop = 2
vim.opt_global.tabstop = 4
```

### 执行 Vimscript

```lua
-- 执行命令
vim.cmd("colorscheme habamax")
vim.cmd.echo("hello")     -- 面向对象风格

-- 调用函数
vim.fn.expand("%")        -- 当前文件名
vim.fn.has("nvim-0.10")   -- 功能检测
vim.call("printf", "hello %s", "world")

-- API 直接调用
vim.api.nvim_get_current_buf()
```

### 按键映射

```lua
vim.keymap.set("n", "<Leader>w", "<cmd>w<cr>", {desc = "保存"})
vim.keymap.del("n", "<Leader>w")
```

---

## 三、子模块分类介绍

### vim.fs -- 文件系统工具

```lua
vim.fs.normalize("~/foo/./bar")          -- "/home/user/foo/bar"
vim.fs.dirname("/a/b/c.lua")             -- "/a/b"
vim.fs.basename("/a/b/c.lua")            -- "c.lua"
vim.fs.ext("/a/b/c.lua")                 -- "lua"
vim.fs.joinpath("/a", "b", "c.lua")      -- "/a/b/c.lua"
vim.fs.root(".", ".git")                -- 找到包含 .git 的父目录
vim.fs.find({"init.lua"}, {upward=true}) -- 向上查找 init.lua
vim.fs.dir("/path", {depth=1}):each(function(name, type) end)
vim.fs.rm("/tmp/old", {recursive=true})  -- 递归删除
```

### vim.iter -- 惰性迭代器管道

类似 Rust 的 Iterator 或 JS 的 Array 方法，但惰性求值：

```lua
-- 从表创建迭代器
vim.iter({1, 2, 3, 4, 5})
  :filter(function(v) return v % 2 == 0 end)   -- 只要偶数
  :map(function(v) return v * 10 end)            -- 乘以 10
  :totable()                                     -- 收集回 table
-- -> {20, 40}

-- 其他常用方法
vim.iter(t):any(fn)        -- 任一满足？
vim.iter(t):all(fn)        -- 全部满足？
vim.iter(t):find(fn)       -- 找第一个
vim.iter(t):fold(0, fn)    -- 归约
vim.iter(t):join(",")      -- 用分隔符连接成字符串
vim.iter(t):enumerate()    -- 带索引遍历
vim.iter(t):skip(2):take(3) -- 跳过2个取3个
vim.iter(t):rev()          -- 反向遍历（仅列表）
vim.iter(t):unique()       -- 去重
```

### vim.ui -- 用户界面

```lua
-- 文本输入
vim.ui.input({prompt = "请输入文件名："}, function(input)
  if input then print("输入了: " .. input) end
end)

-- 选择列表（可被 telescope/fzf 等插件接管）
vim.ui.select({"apple", "banana", "cherry"}, {
  prompt = "选一个水果",
  format_item = function(item) return item end,
}, function(choice)
  if choice then print("选了: " .. choice) end
end)

-- 用系统默认程序打开文件/URL
vim.ui.open("https://neovim.io")
vim.ui.open("~/document.pdf")
```

### vim.version -- 语义化版本

```lua
local v = vim.version.parse("1.2.3")
print(v.major, v.minor, v.patch)   -- 1  2  3

vim.version.lt(v, "1.3.0")         -- true
vim.version.gt(v, "1.0.0")         -- true
vim.version.cmp("1.2.0", "1.3.0")  -- -1

-- 版本范围
local r = vim.version.range("1.0 - 2.0")
r:has(vim.version.parse("1.5"))    -- true

-- 获取当前 Neovim 版本
local nv = vim.version()
print(nv)  -- 0.12.2
```

### vim.system -- 执行系统命令

```lua
-- 同步（等待完成）
local obj = vim.system({"ls", "-la"}, {text = true}):wait()
print(obj.stdout)  -- 命令输出
print(obj.code)    -- 退出码

-- 异步
vim.system({"find", "/"}, {
  text = true,
  stdout = function(err, data) print(data) end,
}, function(obj) print("完成, 退出码:", obj.code) end)
```

### vim.uv -- libUV 绑定（异步 I/O）

底层事件循环接口，支持定时器、TCP/UDP、文件系统监控、多线程等：

```lua
-- 定时器
local timer = vim.uv.new_timer()
timer:start(1000, 0, vim.schedule_wrap(function()
  print("1秒后执行一次")
  timer:close()
end))

-- 文件变更监控
local watcher = vim.uv.new_fs_event()
watcher:start("/path/to/file", {}, vim.schedule_wrap(function(err, fname, status)
  vim.cmd("checktime")
end))
```

> vim.uv 回调里不能直接调 vim.api，需要用 vim.schedule_wrap 包一层。

### vim.regex -- Vim 正则

```lua
local re = vim.regex("foo\\d\\+")
print(re:match_str("hello foo123 bar"))  -- 匹配对象或 nil
print(re:match_line(0, 5))               -- 在某 buffer 第 5 行匹配
```

### vim.re -- LPeg 正则接口

LPeg 是 Lua 的解析表达式语法库，比 PCRE 更强大：

```lua
local pattern = vim.re.compile([[ (foo|bar)\d+ ]])
print(vim.re.match("foo123", pattern))   -- "foo123"
print(vim.re.find("hello bar456", pattern))  -- 返回起止位置
```

---

## 四、其他重要子模块速查

| 子模块 | 用途 |
|--------|------|
| `vim.lsp` | LSP 客户端（跳转、补全、诊断） |
| `vim.diagnostic` | 诊断信息展示和管理 |
| `vim.treesitter` | 语法高亮、折叠、文本对象 |
| `vim.hl` | 高亮控制（on_yank、range） |
| `vim.snippet` | 代码片段展开和跳转 |
| `vim.spell` | 拼写检查 |
| `vim.filetype` | 文件类型检测 |
| `vim.secure` | 信任数据库（安全加载脚本） |
| `vim.base64` | Base64 编解码 |
| `vim.glob` | Glob 模式转 LPeg |
| `vim.net` | HTTP 客户端 |
| `vim.text` | 文本工具（diff、hex 编解码、缩进） |
| `vim.inspector` | 光标位置检查 |
| `vim.pos` / `vim.range` | 位置和范围抽象类型 |
| `vim.uri` | URI 编解码与路径互转 |
| `vim.loader` | 快速模块加载缓存 |
| `vim.ui.img` | 终端内嵌图片显示 |

---

## 五、stdlib 的核心设计思想

1. **全局可用**：vim 模块在 Neovim 启动时自动创建，所有地方都能直接用，不需要 require

2. **渐进式封装**：
   - 底层 C API 通过 vim.api 暴露
   - Vimscript 兼容通过 vim.fn/vim.cmd 暴露
   - 便利封装在上层（vim.keymap、vim.opt、vim.iter 等）

3. **一致性**：函数命名有规律可循
   - tbl_* 操作表
   - list_* 操作列表
   - fs.* 文件系统
   - ui.* 用户交互

4. **可扩展**：许多模块允许插件覆盖行为
   - vim.notify 可被 notification.nvim / noice.nvim 接管
   - vim.ui.select 可被 telescope / fzf-lua 接管
   - vim.ui.input 可被 dressing.nvim 接管

---

**总结**：stdlib 是 Neovim 给 Lua 开发者的一套完整 SDK。写配置时你主要在跟 vim.o / vim.opt / vim.keymap / vim.api.nvim_create_autocmd 打交道；写插件时会用到 vim.fs / vim.iter / vim.system / vim.uv / vim.validate 等更底层的能力。
