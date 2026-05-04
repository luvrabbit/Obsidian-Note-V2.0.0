# vim.pack 完全指南

`vim.pack` 涉及两层含义：
1. **Packages 目录机制** -- Vim/Neovim 传统的插件组织约定
2. **vim.pack 模块** -- Neovim 0.10+ 内置的实验性 Lua 插件管理器

---

## 第一部分：Packages 目录机制

### 目录结构

```
~/.local/share/nvim/site/pack/     <- packpath 根目录
├── my-package/                    <- 包名可任意取
│   ├── start/                     <- 启动时自动加载
│   │   ├── plugin-a/
│   │   │   ├── plugin/foo.vim
│   │   │   ├── autoload/foo.vim
│   │   │   └── doc/foo.txt
│   │   └── plugin-b/
│   │       └── plugin/bar.vim
│   └── opt/                       <- 手动按需加载
│       ├── colorscheme-dark/
│       │   └── colors/dark.vim
│       └── debug-tools/
│           └── plugin/debugger.vim
└── another-package/
    └── start/
        └── plugin-c/
            └── plugin/baz.vim
```

> 注意：里层目录（如 `plugin-a/`）不能省略，直接放 `start/foo.vim` 是无效的。

### start 和 opt 的区别

| 目录 | 加载时机 | 典型用途 |
|------|---------|---------|
| `pack/*/start/*` | 启动后自动加载 | 日常必需插件 |
| `pack/*/opt/*` | 不加载，需 `:packadd 插件名` | colorscheme、条件加载 |

### start 插件（自动加载）

放在 `pack/*/start/*/` 下的插件在 Neovim 启动时自动加载：

1. Nvim 处理完 `init.lua`/`init.vim`
2. 扫描 packpath 下所有 `pack/*/start/*/` 路径
3. 自动 source 每个插件的 `plugin/` 和 `ftdetect/` 文件
4. 插件目录加入 runtime 搜索路径

**注意**：start 目录不在 `:set runtimepath?` 输出中显示，需用 `nvim_list_runtime_paths()` 查看。

**colorscheme 特殊行为**：`:colorscheme` 命令不仅搜索 runtimepath，也搜索 packpath 下所有 start 和 opt 目录。所以 colorscheme 放在 `pack/*/opt/` 下无需先 packadd。

### opt 插件（手动加载）

```vim
:packadd plugin-name
```

`:packadd` 的工作方式：
1. 搜索 packpath 下所有 `pack/*/opt/{name}/` 路径
2. 把找到的目录临时加入 runtimepath 最前面
3. source 该插件的 `plugin/` 和 `ftdetect/` 文件

**在 init.lua 中加载 opt 插件**：
```lua
-- 直接加载
vim.cmd.packadd('foodebug')

-- 条件加载
if vim.fn.has('nvim-0.10') == 1 then
  vim.cmd.packadd('new-feature-plugin')
end
```

**在 init.vim 中**：
```vim
packadd! foodebug    " ! 表示 --noplugin 时不加载
```

### 运行时搜索路径

Nvim 搜索 `:runtime` 文件的顺序：
1. `runtimepath` 中所有目录
2. 所有 `pack/*/start/*` 目录

这意味着 start 插件中的 `syntax/`、`ftplugin/`、`indent/` 等目录自动可用。

### 插件之间的依赖

```
pack/foo/start/
├── plugin-one/plugin/one.vim     <- call foolib#getit()
├── plugin-two/plugin/two.vim     <- call foolib#getit()
└── common-lib/autoload/foolib.vim <- function foolib#getit()
```

这种结构有效，因为所有 start 插件目录一起加入搜索路径，autoload 函数在运行时按需查找。

### 文件放置最佳实践

| 文件类型 | 推荐位置 | 原因 |
|---------|---------|------|
| Colorschemes | `pack/*/opt/*/colors/` | 自动可被 `:colorscheme` 发现 |
| Filetype 插件 | `pack/*/start/` | 需要始终可用 |
| 条件性功能 | `pack/*/opt/` | 只在需要时 packadd |
| 依赖库 | `pack/*/start/lib/autoload/` | autoload 自动搜索 |

### 发布 package 的目录结构

```
start/myplugin/plugin/foo.vim        <- 始终加载
start/myplugin/plugin/bar.vim        <- 始终加载
start/myplugin/autoload/foo.vim      <- 惰性加载
start/myplugin/doc/foo.txt           <- 帮助文档
start/myplugin/doc/tags
opt/myplugin-extra/plugin/extra.vim  <- 可选插件
opt/myplugin-extra/autoload/extra.vim
opt/myplugin-extra/doc/extra.txt
opt/myplugin-extra/doc/tags
```

生成帮助标签：`:helptags path/to/doc`

---

## 第二部分：vim.pack 模块（内置插件管理器）

### 概述

vim.pack 是 Neovim 0.10+ 内置的**实验性** Lua 插件管理器，利用 packages 机制提供声明式 API。

> 状态：实验性，但日常使用已基本稳定。

### 核心设计

**统一存放位置**：所有插件放在 `{data}/site/pack/core/opt/`。data 即 `stdpath('data')`，通常是 `~/.local/share/nvim/`。

```
~/.local/share/nvim/site/pack/core/opt/
├── nvim-treesitter/
├── telescope.nvim/
├── plenary.nvim/
└── ...
```

子目录名即为插件名。

**锁文件**：`~/.config/nvim/nvim-pack-lock.json`

持久化记录每个插件的 src、rev、version。推荐放入 git 实现多机同步。不应手动编辑。

**锁文件策略**：
- 锁文件存在且包含某插件 -> 安装锁文件记录的版本（忽略 spec 的 version）
- 锁文件缺失某插件 -> 根据 spec 的 version 推断版本

**前提条件**：系统安装 git，插件仓库有 semver 格式 tag。

---

### API 详解

#### vim.pack.Spec -- 插件规格

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `src` | string | **是** | Git 仓库 URL |
| `name` | string | 否 | 插件名（用作目录名），默认从 URL 推断 |
| `version` | string/VimVersionRange/nil | 否 | 版本约束 |
| `data` | any | 否 | 自定义数据 |

**version 字段的三种用法**：

```lua
-- 1. nil：跟随默认分支（main 或 master）
{ src = 'https://github.com/user/plugin' }

-- 2. 字符串：指定分支名、标签名或 commit hash
{ src = 'https://github.com/user/plugin', version = 'develop' }
{ src = 'https://github.com/user/plugin', version = 'abc123def' }

-- 3. vim.version.range()：semver 版本范围
{ src = 'https://github.com/user/plugin', version = vim.version.range('1.0') }
{ src = 'https://github.com/user/plugin', version = vim.version.range('>=1.2 <2.0') }
```

---

#### vim.pack.add(specs, opts) -- 添加插件

```lua
vim.pack.add({
  -- 字符串形式：等于 { src = '...' }
  'https://github.com/nvim-lua/plenary.nvim',

  -- 完整 table 形式
  {
    src = 'https://github.com/nvim-treesitter/nvim-treesitter',
    name = 'treesitter',
    version = vim.version.range('>=0.9'),
  },

  { src = 'https://github.com/neovim/nvim-lspconfig' },
  { src = 'https://github.com/hrsh7th/nvim-cmp' },
})
```

**执行流程（每个插件）**：
1. 检查 `{data}/site/pack/core/opt/{name}/` 是否存在
   - 存在且 src 相同 -> 跳过
   - 存在但 src 不同 -> 先删除再重新 clone
2. 不存在 -> blobless clone 下载
3. checkout 到 version 对应的 revision
4. 执行 `:packadd {name}` 使插件可用

安装是**并行执行**的，等待全部完成后继续后续代码。add() 返回后插件模块即可 require()。

**opts 参数**：

| 选项 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `confirm` | boolean | `true` | 首次安装时是否弹确认 |
| `load` | boolean/function | false(启动)/true(运行时) | 是否自动 packadd |

```lua
-- 自定义加载逻辑
vim.pack.add({ 'https://github.com/user/plugin' }, {
  load = function(data)
    if some_condition then
      vim.cmd.packadd(data.spec.name)
    end
  end,
})
```

**URL 简化**：
```lua
-- Lua helper
local gh = function(x) return 'https://github.com/' .. x end
vim.pack.add({ gh('user/plugin1'), gh('user/plugin2') })

-- git insteadOf
-- git config --global url."https://github.com/".insteadOf "gh:"
vim.pack.add({ 'gh:user/plugin1', 'gh:user/plugin2' })
```

---

#### vim.pack.update(names, opts) -- 更新插件

```lua
vim.pack.update()                                   -- 更新所有
vim.pack.update({ 'telescope.nvim' })               -- 更新指定
vim.pack.update(nil, { force = true })              -- 跳过确认直接更新
vim.pack.update(nil, { offline = true })            -- 离线模式
vim.pack.update({ 'plugin' }, { target = 'lockfile' })  -- 回退
```

**opts 参数**：

| 选项 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `force` | boolean | `false` | 跳过确认 buffer |
| `offline` | boolean | `false` | 不拉取远程 |
| `target` | string | `"version"` | 目标版本来源 |

更新日志记录在 `{log}/nvim-pack.log`。

---

#### 确认 Buffer

执行 `update()` 不带 `force = true` 时，弹出特殊 tabpage：

- 每个插件一个 section，显示当前 revision -> 目标 revision + changelog
- `>` 开头 = 将应用的变更，`<` 开头 = 将撤销的变更

| 操作 | 快捷键/命令 |
|------|-----------|
| 确认所有更新 | `:w` |
| 取消所有更新 | `:q` |
| 下一个/上一个插件 | `]]` / `[[` |
| 查看 buffer 结构 | `gO` (LSP documentSymbol) |
| 查看详情 | `K` (LSP hover) |
| 当前插件操作 | `gra` (codeAction) |
| 打开链接 | `gx` (LSP documentLink) |

Code action 菜单：
- **delete**：删除不在 session 中的插件
- **update**：更新到目标版本
- **skip updating**：跳过某个插件

---

#### vim.pack.del(names, opts) -- 删除插件

```lua
vim.pack.del({ 'old-plugin' })
vim.pack.del({ 'plugin' }, { force = true })  -- 强制删除
```

| 选项 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `force` | boolean | `false` | 允许删除当前激活的插件 |

删除前必须先从 init.lua 移除对应 add() 调用。

```lua
-- 批量删除未激活插件
local to_delete = vim.iter(vim.pack.get())
  :filter(function(x) return not x.active end)
  :map(function(x) return x.spec.name end)
  :totable()
vim.pack.del(to_delete)
```

---

#### vim.pack.get(names, opts) -- 获取插件状态

```lua
local all = vim.pack.get()                       -- 所有插件
local info = vim.pack.get({ 'telescope.nvim' })  -- 指定插件
local names = vim.pack.get(nil, { info = false }) -- 仅名字
```

**返回字段**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `spec` | table | 解析后的插件规格 |
| `path` | string | 磁盘路径 |
| `rev` | string | 当前 commit hash |
| `active` | boolean | 是否在当前 session 激活 |
| `branches` | string[] | Git 分支列表（info=true） |
| `tags` | string[] | Git 标签列表（info=true） |

---

### 事件系统

| 事件 | 触发时机 |
|------|---------|
| `PackChangedPre` | 即将变更插件状态前 |
| `PackChanged` | 插件状态变更完成后 |

事件数据字段：`spec`、`path`、`active`、`kind`（"install"/"update"/"delete"）

**post-install/build hook 示例**：
```lua
vim.api.nvim_create_autocmd('PackChanged', {
  callback = function(ev)
    local name, kind = ev.data.spec.name, ev.data.kind
    if name == 'treesitter' and (kind == 'install' or kind == 'update') then
      vim.system({ 'make' }, { cwd = ev.data.path })
    end
    if name == 'my-plugin' and kind == 'update' then
      if not ev.data.active then vim.cmd.packadd('my-plugin') end
      require('my-plugin').after_update()
    end
  end,
})
```

---

### 实用场景

**多机同步**：
```bash
# 主机器提交锁文件
git add ~/.config/nvim/nvim-pack-lock.json
git commit -m "lock" && git push
# 新机器 pull 后启动 nvim，插件自动安装到锁文件记录的版本
```

**冻结版本**：`version = 'commit-hash'`

**切换源**：改 src -> restart -> 旧源自动删除，新源 clone

**回退**：`git checkout HEAD -- nvim-pack-lock.json` -> restart -> `update({target='lockfile'})`

---

### 与第三方对比

| 特性 | vim.pack | Lazy.nvim | packer | vim-plug |
|------|---------|-----------|--------|----------|
| 内置 | **是** | 否 | 否 | 否 |
| 安装 | 零依赖 | 需 bootstrap | 需 bootstrap | 需 bootstrap |
| 锁文件 | 内置 JSON | lazy-lock.json | 无 | 无 |
| 更新 UI | tabpage + LSP | 浮动窗口 | 浮动窗口 | 基础窗口 |
| 版本约束 | semver range | commit/tag/branch | commit/tag/branch | branch |
| 懒加载 | opt + packadd | 丰富策略 | 丰富策略 | 基础 |
| 成熟度 | 实验性 | **稳定** | 停更 | 稳定 |

---

### 局限

1. 实验性 API，未来可能变化
2. 默认分支变更（master->main）不会自动跟随
3. 需要 git，不支持 zip/本地路径
4. 所有插件必须在同一 pack/core/opt/ 下
5. 懒加载只有 opt + packadd 方式

---

### 推荐场景

| 场景 | 推荐 |
|------|------|
| 零依赖内置方案 | vim.pack |
| 丰富懒加载 | Lazy.nvim |
| 稳定生产 | Lazy.nvim |
| 体验新特性 | vim.pack |
