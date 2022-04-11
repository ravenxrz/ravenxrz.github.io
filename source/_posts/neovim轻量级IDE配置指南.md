---
title: Neovim轻量级IDE配置指南
categories: neovim
date: 2022-04-8 22:10:31
tags:
---



## 前言

很久没更新博文，这一个月除了忙着论文，也抽空学了下neovim。在各类ide和vscode中用vim插件已经过了3 4年，常见vim操作已经烂熟于心。一直觉得纯vim/nvim很难用，即使配上插件也难以适用到平日的开发工作。 直到看到[油管大佬](https://www.youtube.com/watch?v=ctH-a-1eUME&list=PLhoH5vyxr6Qq41NFL4GvhFp-WLd5xzIzZ)的视频，重燃些许兴趣，一路跟着配置，感觉neovim已经完全具备常用的开发功能。至少目前的配置效果让我决定先使用neovim做一两个小项目。

**本文算是一个对neovim配置的总结篇**

<!--more-->


目前来说最终配置的插件包含如下80多个，不会每个都详细介绍，仅列出个人觉得最重要和难以配置的插件，比如lsp和dap。

**开始配置前，请给你的终端换成nerd font字体，否则可能有些图标是乱码。**

**我使用的是 jetbrains mono nerd font.**

贴出我个人的配置：https://github.com/ravenxrz/dotfiles/tree/master/nvim

## 1. neovim配置文件结构与基本设置

**neovim版本 0.6.1 及其以上。**

在 `~/.config/nvim` 目录下创建如下目录：

```lua
init.lua							// 初始化
lua
└── user
    ├── colorscheme.lua				// 主题设置相关
    ├── conf/						// 常用插件配置
    ├── dap/						// debugger 设置
    ├── keymaps.lua					// 快捷键设置
    ├── lsp/						// lsp 设置
    ├── options.lua					// 常用neovim
    ├── plugins.lua					// 插件管理
    └── utils.lua					// 可选
```

init.lua 脚本和lua目录同级别，打开Init.lua脚本，写入如下配置：

```lua
require "user.options"
require "user.keymaps"
require "user.plugins"
require "user.colorscheme"
require "user.utils"

require "user.conf"
require "user.lsp"
require "user.dap"
```

**！！！另外在每个目录下都建立一个init.lua文件！！！**

### 基本设置

首先配置 optoins.lua, 和vim不同，neovim支持全lua脚本配置，所以看起来就非常舒心了。

```lua
local options = {
  backup = false,                          -- creates a backup file
  clipboard = "unnamedplus",               -- allows neovim to access the system clipboard
  cmdheight = 2,                           -- more space in the neovim command line for displaying messages
  completeopt = { "menuone", "noselect" }, -- mostly just for cmp
  conceallevel = 0,                        -- so that `` is visible in markdown files
  fileencoding = "utf-8",                  -- the encoding written to a file
  hlsearch = true,                         -- highlight all matches on previous search pattern
  ignorecase = true,                       -- ignore case in search patterns
  mouse = "a",                             -- allow the mouse to be used in neovim
  pumheight = 10,                          -- pop up menu height
  showmode = false,                        -- we don't need to see things like -- INSERT -- anymore
  showtabline = 2,                         -- always show tabs
  smartcase = true,                        -- smart case
  smartindent = true,                      -- make indenting smarter again
  splitbelow = true,                       -- force all horizontal splits to go below current window
  splitright = true,                       -- force all vertical splits to go to the right of current window
  swapfile = false,                        -- creates a swapfile
  termguicolors = true,                    -- set term gui colors (most terminals support this)
  timeoutlen = 500,                        -- time to wait for a mapped sequence to complete (in milliseconds)
  undofile = true,                         -- enable persistent undo
  updatetime = 300,                        -- faster completion (4000ms default)
  writebackup = false,                     -- if a file is being edited by another program (or was written to file while editing with another program), it is not allowed to be edited
  expandtab = true,                        -- convert tabs to spaces
  shiftwidth = 2,                          -- the number of spaces inserted for each indentation
  tabstop = 2,                             -- insert 2 spaces for a tab
  cursorline = true,                       -- highlight the current line
  cursorcolumn = false,                    -- cursor column.
  number = true,                           -- set numbered lines
  relativenumber = false,                  -- set relative numbered lines
  numberwidth = 4,                         -- set number column width to 2 {default 4}
  signcolumn = "yes",                      -- always show the sign column, otherwise it would shift the text each time
  wrap = false,                            -- display lines as one long line
  scrolloff = 8,                           -- is one of my fav
  sidescrolloff = 8,
  guifont = "monospace:h17",               -- the font used in graphical neovim applications
  foldmethod = "expr",                     -- fold with nvim_treesitter
  foldexpr = "nvim_treesitter#foldexpr()", 
  foldenable = false,                      -- no fold to be applied when open a file
  foldlevel = 99,                          -- if not set this, fold will be everywhere
}

vim.opt.shortmess:append "c"

for k, v in pairs(options) do
  vim.opt[k] = v
end

vim.cmd "set whichwrap+=<,>,[,],h,l"
vim.cmd [[set iskeyword+=-]]
vim.cmd [[set formatoptions-=cro]] -- TODO: this doesn't seem to work


-- WSL yank support
vim.cmd [[
let s:clip = '/mnt/c/Windows/System32/clip.exe' 
if executable(s:clip)
    augroup WSLYank
        autocmd!
        autocmd TextYankPost * if v:event.operator ==# 'y' | call system(s:clip, @0) | endif
    augroup END
endif
]]
```

这个配置有详细的注释，值得说的有两点：

1. fold相关，这个依赖后续的treesitter插件，现在可先注释掉， 否则重启nvim可能会报错
2. wsl相关，如果你在使用windows开发，想要通过 `y` 拷贝文本内容至系统剪切板，则需要加末尾处的代码。

### 快捷键配置

打开keymaps.lua,  根据个人习惯设置快捷键，我的部分配置如下：

```lua
local opts = { noremap = true, silent = true }

local term_opts = { silent = true }

-- Shorten function name
local keymap = vim.api.nvim_set_keymap

--Remap ; as leader key
keymap("", ";", "<Nop>", opts)
vim.g.mapleader = ";"
vim.g.maplocalleader = ";"

-- Modes normal_mode = "n",
--   insert_mode = "i",
--   visual_mode = "v",
--   visual_block_mode = "x",
--   term_mode = "t", command_mode = "c",

-- Normal --
-- Better window navigation
keymap("n", "<C-h>", "<C-w>h", opts)
keymap("n", "<C-j>", "<C-w>j", opts)
keymap("n", "<C-k>", "<C-w>k", opts)
keymap("n", "<C-l>", "<C-w>l", opts)
-- NOTE: require winshit plugin
keymap("n", "<C-W>m", ":WinShift<cr>", opts)

-- FileExpolre
keymap("n", ";e", ":NvimTreeToggle<cr>", opts)
keymap("n", ";f", ":NvimTreeFindFile<cr>", opts)
-- no highlight
keymap("n", "<leader>l", ":nohl<cr>", opts)
-- save buffer
keymap("n", "<leader>w", ":w<cr>", opts)
-- quite buffer


-- exit cur window
keymap("n", "<leader>q", ":q<cr>", opts)
-- delete cur buffer
keymap("n", "<leader>d", ":Bdelete<cr>", opts)
-- exit whole program 
-- keymap("n", "ZZ", ":lua require('user.utils').SaveAndExit()<cr>", opts)
-- remap macro record key
keymap("n", "Q", "q", opts)
-- cancel q
keymap("n", "q", "<Nop>", opts)

-- center cursor
keymap("n", "n", "nzzzv", opts)
keymap("n", "N", "Nzzzv", opts)
keymap("n", "J", "mzJ`z", opts)
-- keymap("n", "j", "jzz", opts)
-- keymap("n", "k", "kzz", opts)

-- Resize with arrows
keymap("n", "<C-Up>", ":resize -2<CR>", opts)
keymap("n", "<C-Down>", ":resize +2<CR>", opts)
keymap("n", "<C-Left>", ":vertical resize -2<CR>", opts)
keymap("n", "<C-Right>", ":vertical resize +2<CR>", opts)

-- Navigate buffers
-- keymap("n", "R", ":bnext<CR>", opts)
-- keymap("n", "E", ":bprevious<CR>", opts)
-- NOTE: E/R navigation needs  'bufferline' plugin
keymap("n", "R", ":BufferLineCycleNext<CR>", opts)
keymap("n", "E", ":BufferLineCyclePrev<CR>", opts)

-- Navigate line
keymap("n", "H", "^", opts)
keymap("n", "L", "$", opts)
keymap("v", "H", "^", opts)
keymap("v", "L", "$", opts)

-- Move text up and down
keymap("n", "<A-j>", "<Esc>:m .+1<CR>==gi", opts)
keymap("n", "<A-k>", "<Esc>:m .-2<CR>==gi", opts)

-- Insert --
-- Press jk fast to enter
keymap("i", "jk", "<ESC>", opts)

-- Visual --
-- Stay in indent mode
keymap("v", "<", "<gv", opts)
keymap("v", ">", ">gv", opts)

-- Move text up and down
-- keymap("v", "<A-j>", ":m .+1<CR>==", opts)
-- keymap("v", "<A-k>", ":m .-2<CR>==", opts)
keymap("v", "p", '"_dP', opts)

```

keymap代表的是快捷键设置函数：

1. 第一个参数 “n”, “i”, “v” 等分别代表了vim的normal、insert、visual模式
2. 第二个参数代表要映射到的key
3. 第三个参数代表要被替换的key
4. 第四个参数可忽略

### 主题配置

可通过:colorscheme <Tab> 查看目前系统的主题， 若要将主题持久化，可打开 colorscheme.lua 文件，填入如下内容

```lua
local colorscheme = "darkplus"

local status_ok, _ = pcall(vim.cmd, "colorscheme " .. colorscheme)
if not status_ok then
  vim.notify("colorscheme " .. colorscheme .. " not found!")
  return
end

if colorscheme == "onedark" then
  require "user.conf.onedark"
end
```

若要修改主题名， 直接修改 colorscheme变量即可。

## 2. 插件管理

推荐使用packer.nvim进行插件管理

执行如下代码安装packer:

```lua
git clone --depth 1 https://github.com/wbthomason/packer.nvim\
 ~/.local/share/nvim/site/pack/packer/start/packer.nvim
```

然后打开plugins.lua文件：

```lua
local fn = vim.fn

-- Automatically install packer
local install_path = fn.stdpath "data" .. "/site/pack/packer/start/packer.nvim"
if fn.empty(fn.glob(install_path)) > 0 then
  PACKER_BOOTSTRAP = fn.system {
    "git",
    "clone",
    "--depth",
    "1",
    "https://github.com/wbthomason/packer.nvim",
    install_path,
  }
  print "Installing packer close and reopen Neovim..."
  vim.cmd [[packadd packer.nvim]]
end

-- Autocommand that reloads neovim whenever you save the plugins.lua file
vim.cmd [[
  augroup packer_user_config
    autocmd!
    autocmd BufWritePost plugins.lua source <afile> | PackerSync
  augroup end
]]

-- Use a protected call so we don't error out on first use
local status_ok, packer = pcall(require, "packer")
if not status_ok then
  return
end

-- Have packer use a popup window
packer.init {
  display = {
    -- open_fn = function()
    --   return require("packer.util").float { border = "rounded" }
    -- end,
  },
}


--  useage
-- use {
--   "myusername/example",        -- The plugin location string
--   -- The following keys are all optional
--   disable = boolean,           -- Mark a plugin as inactive
--   as = string,                 -- Specifies an alias under which to install the plugin
--   installer = function,        -- Specifies custom installer. See "custom installers" below.
--   updater = function,          -- Specifies custom updater. See "custom installers" below.
--   after = string or list,      -- Specifies plugins to load before this plugin. See "sequencing" below
--   rtp = string,                -- Specifies a subdirectory of the plugin to add to runtimepath.
--   opt = boolean,               -- Manually marks a plugin as optional.
--   branch = string,             -- Specifies a git branch to use
--   tag = string,                -- Specifies a git tag to use. Supports "*" for "latest tag"
--   commit = string,             -- Specifies a git commit to use
--   lock = boolean,              -- Skip updating this plugin in updates/syncs. Still cleans.
--   run = string, function, or table, -- Post-update/install hook. See "update/install hooks".
--   requires = string or list,   -- Specifies plugin dependencies. See "dependencies".
--   rocks = string or list,      -- Specifies Luarocks dependencies for the plugin
--   config = string or function, -- Specifies code to run after this plugin is loaded.
--   -- The setup key implies opt = true
--   setup = string or function,  -- Specifies code to run before this plugin is loaded.
--   -- The following keys all imply lazy-loading and imply opt = true
--   cmd = string or list,        -- Specifies commands which load this plugin. Can be an autocmd pattern.
--   ft = string or list,         -- Specifies filetypes which load this plugin.
--   keys = string or list,       -- Specifies maps which load this plugin. See "Keybindings".
--   event = string or list,      -- Specifies autocommand events which load this plugin.
--   fn = string or list          -- Specifies functions which load this plugin.
--   cond = string, function, or list of strings/functions,   -- Specifies a conditional test to load this plugin
--   module = string or list      -- Specifies Lua module names for require. When requiring a string which starts
--                                -- with one of these module names, the plugin will be loaded.
--   module_pattern = string/list -- Specifies Lua pattern of Lua module names for require. When
--   requiring a string which matches one of these patterns, the plugin will be loaded.
-- }

-- Install your plugins here
return packer.startup(function(use)	
  -- My plugins here
  use "wbthomason/packer.nvim" -- Have packer manage itself
  use "nvim-lua/popup.nvim" -- An implementation of the Popup API from vim in Neovim
  use "nvim-lua/plenary.nvim" -- Useful lua functions used ny lots of plugins


  -- Automatically set up your configuration after cloning packer.nvim
  -- Put this at the end after all plugins
  if PACKER_BOOTSTRAP then
    require("packer").sync()
  end
end)

```

插件写在后续添加插件时， 写在 packer.startup()函数内，然后保存即可，每次保存 plugins.lua 文件时，将会自动更新插件列表。

packer常见的命令包括：

![image-20220408175005981](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20220408175005981.png)

**通常只用使用PackerSync(插件同步，下载新插件+更新插件), PackerInstall (安装新插件), PackerStatus(查看目前插件的状态)**

## 3. 主题插件 + 语法高亮

好看的主题 + 语法高亮是写代码的基础，在白花花的一片代码中寻找某个符号是难以接受的。

推荐两个主题：

```
  use "lunarvim/darkplus.nvim"			-- 类似vscode的主题
  use "navarasu/onedark.nvim"			-- onedark，atom的经典主题, onedark又包含很多子主题，比如dark，light，warm等
```

效果：

darkplus

![image-20220411102359755](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20220411102359755.png)

onedark 的dark子主题:

![image-20220411102445355](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20220411102445355.png)

其中onedark可进行详细配置, 建立config/onedark.lua文件

```lua
-- TODO: 改变theme不生效
local status_ok, onedark = pcall(require, "onedark")
if not status_ok then
  vim.notify("onedark theme not found!")
  return
end

-- NOTE: if use 'light' theme, you  should change both backgournd and style to 'light'
vim.o.background='dark'
-- vim.o.background='light'
onedark.setup {
  -- Main options --
  style = 'dark', -- Default theme style. Choose between 'dark', 'darker', 'cool', 'deep', 'warm', 'warmer' and 'light'
  transparent = false,  -- Show/hide background
  term_colors = true, -- Change terminal color as per the selected theme style
  ending_tildes = false, -- Show the end-of-buffer tildes. By default they are hidden
  -- toggle theme style ---
  toggle_style_list = {'light', 'dark', 'darker', 'cool', 'deep', 'warm', 'warmer'}, -- List of styles to toggle between
  toggle_style_key = '<leader>ts', -- Default keybinding to toggle

  -- Change code style ---
  -- Options are italic, bold, underline, none
  -- You can configure multiple style with comma seperated, For e.g., keywords = 'italic,bold'
  code_style = {
    comments = 'italic',
    keywords = 'none',
    functions = 'bold',
    strings = 'none',
    variables = 'none'
  },

  -- Custom Highlights --
  colors = {}, -- Override default colors
  highlights = {} -- Override highlight groups
}

onedark.load()

```



**语法高亮采用treesitter插件**，该插件非常强大，语法高亮只是其中一个功能。

```
  -- Treesittetr
  use {
    "nvim-treesitter/nvim-treesitter",
    run = ":TSUpdate",
  }
  use "nvim-treesitter/nvim-treesitter-textobjects"  -- enhance texetobject selection
  use "romgrk/nvim-treesitter-context"  -- show class/function at top
  use "andymass/vim-matchup"
```

建立 config/treesitter.lua文件, 填写如下内容

```
local status_ok, configs = pcall(require, "nvim-treesitter.configs")
if not status_ok then
  vim.notify("treesitter not found!")
  return
end

configs.setup {
  ensure_installed = {"cpp", "lua", "c", "python", "go"}, -- one of "all", "maintained" (parsers with maintainers), or a list of languages
  sync_install = false, -- install languages synchronously (only applied to `ensure_installed`)
  ignore_install = { "" }, -- List of parsers to ignore installing
  autopairs = {
    enable = true,
  },
  highlight = {
    enable = true, -- false will disable the whole extension
    disable = { "" }, -- list of language that will be disabled
    additional_vim_regex_highlighting = true,
  },
  indent = { enable = true, disable = { "yaml" } },
  context_commentstring = {
    enable = true,
    enable_autocmd = false,
  },

  -- textobjects extension settings
  -- https://github.com/nvim-treesitter/nvim-treesitter-textobjects
  textobjects = {
    select = {
      enable = true,
      -- Automatically jump forward to textobj, similar to targets.vim
      lookahead = true,
      keymaps = {
        -- You can use the capture groups defined in textobjects.scm
        ["af"] = "@function.outer",
        ["if"] = "@function.inner",
        ["ac"] = "@class.outer",
        ["ic"] = "@class.inner",
      },
    },
    move = {
      enable = true,
      set_jumps = true, -- whether to set jumps in the jumplist
      goto_next_start = {
        ["]]"] = "@function.outer",
        -- ["]]"] = "@class.outer",
      },
      -- goto_next_end = {
      --   ["jF"] = "@function.outer",
      --   ["]["] = "@class.outer",
      -- },
      goto_previous_start = {
        ["[["] = "@function.outer",
        -- ["[["] = "@class.outer",
      },
      -- goto_previous_end = {
      --   ["kF"] = "@function.outer",
      --   ["[]"] = "@class.outer",
      -- },
    },
    lsp_interop = {
      enable = true,
      border = 'none',
      peek_definition_code = {
        ["<leader>df"] = "@function.outer",
        ["<leader>dF"] = "@class.outer",
      },
    },
  },
  -- matchup plugins
  -- https://github.com/andymass/vim-matchup
  matchup = {
    enable = true,              -- mandatory, false will disable the whole extension
    -- disable = { "c", "ruby" },  -- optional, list of language that will be disabled
    -- [options]
  },
}
```

对需要语法高亮的语言进行配置：

```
  ensure_installed = {"cpp", "lua", "c", "python", "go"}, -- one of "all", "maintained" (parsers with maintainers), or a list of languages
```

有兴趣的朋友还可以看看textobjects配置块，可以提供更强大的移动能力，比如可以通过 “]]”移动到下一个函数。

来看下配置前后的对比：

配置前：

![image-20220411103654262](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20220411103654262.png)

配置后：

![image-20220411103616348](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20220411103616348.png)

为了让配置生效，需要在config/init.lua下添加：

```
require "user.conf.treesitter"
require "user.conf.treesitter-context"		-- 这部分配置文章未贴出，参考github即可
require "user.conf.vim-match"				-- 这部分配置文章贴出，参考github即可
```



## 4. 代码提示+LSP

写代码的另一个重点是需要有代码提示，这部分配置相对复杂，需要依赖很多插件插件。 

> 不了解什么是LSP的，可以先了解下。

首先cmp插件，cmp提供一个补全框架，然后可以通过多个补全源来生成补全内容。补全源可以来自LSP， 可以来自代码片段，可以来自命令行，可以来自路径等等。

比如， 我输入 “/” ，就可以补全根目录下路径：

![image-20220411104136614](D:\坚果云同步\图库\image-20220411104136614.png)

比如说输入vim命令，可以自动补全相关命令：

![image-20220411104204256](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20220411104204256.png)

在特定语言里面，可以根据LSP补全。

所以，首先我们要配置的是cmp框架：

```lua
  use "hrsh7th/nvim-cmp" -- The completion plugin
  use "hrsh7th/cmp-buffer" -- buffer completions
  use "hrsh7th/cmp-path" -- path completions
  use "hrsh7th/cmp-cmdline" -- cmdline completions
  use "saadparwaiz1/cmp_luasnip" -- snippet completions
  use "hrsh7th/cmp-nvim-lsp"	-- lsp support
  use "hrsh7th/cmp-nvim-lua"	-- neovim api
  -- snippets
  use "L3MON4D3/LuaSnip" --snippet engine
  use "rafamadriz/friendly-snippets" -- a bunch of snippets to use
```

建立 config/cmp.lua 文件,

```lua
local cmp_status_ok, cmp = pcall(require, "cmp")
if not cmp_status_ok then
  vim.notify("cmp not found!")
  return
end

local snip_status_ok, luasnip = pcall(require, "luasnip")
if not snip_status_ok then
  vim.notify("luasnip not found!")
  return
end

require("luasnip.loaders.from_vscode").lazy_load()    -- load freindly-snippets
require("luasnip.loaders.from_vscode").load({ paths = { -- load custom snippets
  vim.fn.stdpath("config") .. "/my-snippets"
} }) -- Load snippets from my-snippets folder

local check_backspace = function()
  local col = vim.fn.col "." - 1
  return col == 0 or vim.fn.getline("."):sub(col, col):match "%s"
end

--   פּ ﯟ   some other good icons
local kind_icons = {
  Text = "",
  Method = "m",
  Function = "",
  Constructor = "",
  Field = "",
  Variable = "",
  Class = "",
  Interface = "",
  Module = "",
  Property = "",
  Unit = "",
  Value = "",
  Enum = "",
  Keyword = "",
  Snippet = "",
  Color = "",
  File = "",
  Reference = "",
  Folder = "",
  EnumMember = "",
  Constant = "",
  Struct = "",
  Event = "",
  Operator = "",
  TypeParameter = "",
}
-- find more here: https://www.nerdfonts.com/cheat-sheet

cmp.setup {
  snippet = {
    expand = function(args)
      luasnip.lsp_expand(args.body) -- For `luasnip` users.
    end,
  },
  mapping = {
    ["<C-k>"] = cmp.mapping.select_prev_item(),
		["<C-j>"] = cmp.mapping.select_next_item(),
    ['<C-b>'] = cmp.mapping(cmp.mapping.scroll_docs(-4), { 'i', 'c' }),
    ['<C-f>'] = cmp.mapping(cmp.mapping.scroll_docs(4), { 'i', 'c' }),
    ['<C-Space>'] = cmp.mapping(cmp.mapping.complete(), { 'i', 'c' }),
    ["<C-Space>"] = cmp.mapping(cmp.mapping.complete(), { "i", "c" }),
    ["<C-y>"] = cmp.config.disable, -- Specify `cmp.config.disable` if you want to remove the default `<C-y>` mapping.
    ["<C-e>"] = cmp.mapping {
      i = cmp.mapping.abort(),
      c = cmp.mapping.close(),
    },
    -- Accept currently selected item. If none selected, `select` first item.
    -- Set `select` to `false` to only confirm explicitly selected items.
    ["<CR>"] = cmp.mapping.confirm { select = true },
    ["<Tab>"] = cmp.mapping(function(fallback)
      if cmp.visible() then
        cmp.select_next_item()
      elseif luasnip.expandable() then
        luasnip.expand()
      elseif luasnip.expand_or_jumpable() then
        luasnip.expand_or_jump()
      elseif check_backspace() then
        fallback()
      else
        fallback()
      end
    end, {
      "i",
      "s",
    }),
    ["<S-Tab>"] = cmp.mapping(function(fallback)
      if cmp.visible() then
        cmp.select_prev_item()
      elseif luasnip.jumpable(-1) then
        luasnip.jump(-1)
      else
        fallback()
      end
    end, {
      "i",
      "s",
    }),
  },
  formatting = {
    fields = { "kind", "abbr", "menu" },
    format = function(entry, vim_item)
      -- Kind icons
      vim_item.kind = string.format("%s", kind_icons[vim_item.kind])
      -- vim_item.kind = string.format('%s %s', kind_icons[vim_item.kind], vim_item.kind) -- This concatonates the icons with the name of the item kind
      vim_item.menu = ({
        nvim_lsp = "[LSP]",
        nvim_lua = "[NVIM_LUA]",
        luasnip = "[Snippet]",
        buffer = "[Buffer]",
        path = "[Path]",
      })[entry.source.name]
      return vim_item
    end,
  },
  sources = {
    { name = "nvim_lsp" },
    { name = "nvim_lua" },
    { name = "luasnip" },
    { name = "buffer" },
    { name = "path" },
  },
  confirm_opts = {
    behavior = cmp.ConfirmBehavior.Replace,
    select = false,
  },
  documentation = {
    border = { "╭", "─", "╮", "│", "╯", "─", "╰", "│" },
  },
  experimental = {
    ghost_text = false,
    native_menu = false,
  },
}

-- Use buffer source for `/` (if you enabled `native_menu`, this won't work anymore).
cmp.setup.cmdline('/', {
  sources = {
    { name = 'buffer' }
  }
})

-- Use cmdline & path source for ':' (if you enabled `native_menu`, this won't work anymore).
cmp.setup.cmdline(':', {
  sources = cmp.config.sources({
    { name = 'cmdline' }
  }, {
    { name = 'path' }
  })
})

```

这里面配置的东西有很多，比如从哪儿加载代码片段（ lazy_load()那部分)

代码提示的图标样式，代码补全提示窗的快捷键（如 ctrl+j 和 ctrl+k 分配代表向下选择和向上选择).

代码提示的格式formatting代码处。

代码提示的源有哪些， sources代码处.

有了这些配置，现在应该可以补全buffer、cmdline和path。但是还需要配置lsp才可以真正写代码：

添加如下插件：

```lua
  -- LSP
  use "neovim/nvim-lspconfig" -- enable LSP
  use "williamboman/nvim-lsp-installer" -- simple to use language server installer
  use "kosayoda/nvim-lightbulb"  -- code action
  use "ray-x/lsp_signature.nvim"  -- show function signature when typing
```

建立 lsp/init.lua 文件：

```lua
local status_ok, _ = pcall(require, "lspconfig")
if not status_ok then
	return
end

require("user.lsp.lsp-installer")
require("user.lsp.handlers").setup()
-- require("user.lsp.null-ls")
-- require("user.lsp.lsp-utils")
```

建立 lsp/lsp-installer.lua 文件：

```lua
local status_ok, lsp_installer = pcall(require, "nvim-lsp-installer")
if not status_ok then
  vim.notify("nvim-lspconfig not found!")
	return
end

-- Register a handler that will be called for all installed servers.
-- Alternatively, you may also register handlers on specific server instances instead (see example below).
lsp_installer.on_server_ready(function(server)
	local opts = {
		on_attach = require("user.lsp.handlers").on_attach,
		capabilities = require("user.lsp.handlers").capabilities,
	}

	-- This setup() function is exactly the same as lspconfig's setup function.
	-- Refer to https://github.com/neovim/nvim-lspconfig/blob/master/doc/server_configurations.md
	server:setup(opts)
end)
```

建立 lsp/signature.lua 文件：

```lua
require "lsp_signature".setup({
  debug = false, -- set to true to enable debug logging
  log_path = vim.fn.stdpath("cache") .. "/lsp_signature.log", -- log dir when debug is on
  -- default is  ~/.cache/nvim/lsp_signature.log
  verbose = false, -- show debug line number

  bind = true, -- This is mandatory, otherwise border config won't get registered.
  -- If you want to hook lspsaga or other signature handler, pls set to false
  doc_lines = 10, -- will show two lines of comment/doc(if there are more than two lines in doc, will be truncated);
  -- set to 0 if you DO NOT want any API comments be shown
  -- This setting only take effect in insert mode, it does not affect signature help in normal
  -- mode, 10 by default

  floating_window = true, -- show hint in a floating window, set to false for virtual text only mode

  floating_window_above_cur_line = true, -- try to place the floating above the current line when possible Note:
  -- will set to true when fully tested, set to false will use whichever side has more space
  -- this setting will be helpful if you do not want the PUM and floating win overlap

  floating_window_off_x = 1, -- adjust float windows x position.
  floating_window_off_y = 1, -- adjust float windows y position.


  fix_pos = false,  -- set to true, the floating window will not auto-close until finish all parameters
  hint_enable = true, -- virtual hint enable
  hint_prefix = "🐼 ",  -- Panda for parameter
  hint_scheme = "String",
  hi_parameter = "LspSignatureActiveParameter", -- how your parameter will be highlight
  max_height = 12, -- max height of signature floating_window, if content is more than max_height, you can scroll down
  -- to view the hiding contents
  max_width = 80, -- max_width of signature floating_window, line will be wrapped if exceed max_width
  handler_opts = {
    border = "rounded"   -- double, rounded, single, shadow, none
  },

  always_trigger = false, -- sometime show signature on new line or in middle of parameter can be confusing, set it to false for #58

  auto_close_after = nil, -- autoclose signature float win after x sec, disabled if nil.
  extra_trigger_chars = {}, -- Array of extra characters that will trigger signature completion, e.g., {"(", ","}
  zindex = 200, -- by default it will be on top of all floating windows, set to <= 50 send it to bottom

  padding = '', -- character to pad on left and right of signature can be ' ', or '|'  etc

  transparency = nil, -- disabled by default, allow floating win transparent value 1~100
  shadow_blend = 36, -- if you using shadow as border use this set the opacity
  shadow_guibg = 'Black', -- if you using shadow as border use this set the color e.g. 'Green' or '#121315'
  timer_interval = 200, -- default timer check interval set to lower value if you want to reduce latency
  toggle_key = nil -- toggle signature on and off in insert mode,  e.g. toggle_key = '<M-x>'
})

```

**建立 lsp/handlers.lua 文件**， 这是最重要的文件：

```lua
local M = {}

-- TODO: backfill this to template
M.setup = function()
  local signs = {
    { name = "DiagnosticSignError", text = "" },
    { name = "DiagnosticSignWarn", text = "" },
    { name = "DiagnosticSignHint", text = "" },
    { name = "DiagnosticSignInfo", text = "" },
  }

  for _, sign in ipairs(signs) do
    vim.fn.sign_define(sign.name, { texthl = sign.name, text = sign.text, numhl = "" })
  end

  local config = {
    -- disable virtual text
    virtual_text = true,
    -- show signs
    signs = {
      active = signs,
    },
    update_in_insert = true,
    underline = true,
    severity_sort = true,
    float = {
      focusable = false,
      style = "minimal",
      border = "rounded",
      source = "always",
      header = "",
      prefix = "",
    },
  }

  vim.diagnostic.config(config)

  vim.lsp.handlers["textDocument/hover"] = vim.lsp.with(vim.lsp.handlers.hover, {
    border = "rounded",
  }) vim.lsp.handlers["textDocument/signatureHelp"] = vim.lsp.with(vim.lsp.handlers.signature_help, {
    border = "rounded",
  })
end

local function lsp_highlight_document(client)
  -- Set autocommands conditional on server_capabilities
  if client.resolved_capabilities.document_highlight then
    vim.api.nvim_exec(
      [[
      augroup lsp_document_highlight
        autocmd! * <buffer>
        autocmd CursorHold <buffer> lua vim.lsp.buf.document_highlight()
        autocmd CursorMoved <buffer> lua vim.lsp.buf.clear_references()
      augroup END
    ]],
      false
    )
  end
end

local function lsp_keymaps(bufnr)
  local opts = { noremap = true, silent = true }
  vim.api.nvim_buf_set_keymap(bufnr, "n", "gd", "<cmd>lua vim.lsp.buf.declaration()<CR>", opts)
  vim.api.nvim_buf_set_keymap(bufnr, "n", "<leader>t", "<cmd>lua vim.lsp.buf.definition()<CR>", opts)
  vim.api.nvim_buf_set_keymap(bufnr, "n", "gh", "<cmd>lua vim.lsp.buf.hover()<CR>", opts)
  -- vim.api.nvim_buf_set_keymap(bufnr, "n", "gi", "<cmd>lua vim.lsp.buf.implementation()<CR>", opts)
  -- vim.api.nvim_buf_set_keymap(bufnr, "n", "<C-k>", "<cmd>lua vim.lsp.buf.signature_help()<CR>", opts)
  vim.api.nvim_buf_set_keymap(bufnr, "n", "<leader>rn", "<cmd>lua vim.lsp.buf.rename()<CR>", opts)
  -- vim.api.nvim_buf_set_keymap(bufnr, "n", "<leader>u", "<cmd>lua vim.lsp.buf.references()<CR>", opts)
  vim.api.nvim_buf_set_keymap(bufnr, "n", "<A-cr>", "<cmd>lua vim.lsp.buf.code_action()<CR>", opts)
  -- vim.api.nvim_buf_set_keymap(bufnr, "n", "<leader>f", "<cmd>lua vim.diagnostic.open_float()<CR>", opts)
  vim.api.nvim_buf_set_keymap(bufnr, "n", "<leader>dj", '<cmd>lua vim.diagnostic.goto_prev({ border = "rounded" })<CR>', opts)
  vim.api.nvim_buf_set_keymap(bufnr, "n", "<leader>dk", '<cmd>lua vim.diagnostic.goto_next({ border = "rounded" })<CR>', opts)
  vim.api.nvim_buf_set_keymap(bufnr, "n", "gl", '<cmd>lua vim.diagnostic.open_float()<CR>', opts)
  vim.api.nvim_buf_set_keymap(bufnr, "n", "<leader>dq", "<cmd>lua vim.diagnostic.setloclist()<CR>", opts)
  vim.cmd [[ command! Format execute 'lua vim.lsp.buf.formatting()' ]]
end

M.on_attach = function(client, bufnr)
  -- client.resolved_capabilities.document_formatting = false -- disable all server formating capabilities, use null-ls instead
  -- if client.name == "tsserver" or client.name == "clangd" then
  -- end
  lsp_keymaps(bufnr)
  lsp_highlight_document(client)

  -- add outline support for evey lanuage
  -- require("aerial").on_attach(client, bufnr)

  require "lsp_signature".on_attach()
end

local capabilities = vim.lsp.protocol.make_client_capabilities()

local status_ok, cmp_nvim_lsp = pcall(require, "cmp_nvim_lsp")
if not status_ok then
  return
end

M.capabilities = cmp_nvim_lsp.update_capabilities(capabilities)
capabilities.offsetEncoding = { "utf-16" }
return M
```

基本上只用在 lsp_keymaps 函数里面修改相应快捷键。 额外关注 `virtual_text` 选项，如果为true，诊断信息将直接显示在代码后面：

![image-20220411105635422](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20220411105635422.png)

为false则只有一个图标。

至此，所有lsp配置完成，但是如果你打开代码，依然会发现没有代码提示，这是因为我们还需要安装 lsp server. 不过安装就很简单了，执行 LspInstallInfo, 安装需要的语言服务器即可。移动到对应项目，按i键即可一键安装。

![image-20220411105903925](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20220411105903925.png)

如上，c++语言使用clangd，golang使用gopls， python使用pylsp等等。

**额外说一句，对于c++语言如果想使用clang-tidy，可看我github上的 lsp/settings/clangd.lua 文件是怎么配的，配完后，还需要修改lsp-installer.lua文件。**

到现在，已经可以用neovim写代码了。下面就是对界面、其它功能的各种增强了。

## 5. 代码调试 - DAP

一直以来，终端的调试始终比ide差很多，不能像ide那样点点点，比如c/c++，基本上只能使用gdb或者cgdb等工具来调试。直到发现了dap相关的插件。效果如下：

![image-20220411110509473](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20220411110509473.png)

基本上和vscode的调试界面一致。但是用起来肯定是更快捷的。

安装插件如下：

```lua
  -- Debugger
  use "ravenxrz/DAPInstall.nvim"   -- help us install several debuggers
  use "ravenxrz/nvim-dap"
  use "theHamsta/nvim-dap-virtual-text"
  use "rcarriga/nvim-dap-ui"
  use "nvim-telescope/telescope-dap.nvim"
```

新建 dap/dap-ui.lua 文件：

```lua
local status_ok, dapui = pcall(require, 'dapui')
if not status_ok then
  vim.notify("dapui not found")
  return
end

dapui.setup ({
  icons = { expanded = "▾", collapsed = "▸" },
  mappings = {
    -- Use a table to apply multiple mappings
    expand = { "o", "<2-LeftMouse>", "<CR>" },
    open = "O",
    remove = "d",
    edit = "e",
    repl = "r",
    toggle = "t",
  },
  sidebar = {
    -- You can change the order of elements in the sidebar
    elements = {
      -- Provide as ID strings or tables with "id" and "size" keys
      {
        id = "scopes",
        size = 0.35, -- Can be float or integer > 1
      },
      { id = "stacks", size = 0.35 },
      { id = "watches", size = 0.15 },
      { id = "breakpoints", size = 0.15 },
    },
    size = 40,
    position = "left", -- Can be "left", "right", "top", "bottom"
  },
  tray = {
    elements = { "repl" },
    size = 5,
    position = "bottom", -- Can be "left", "right", "top", "bottom"
  },
  floating = {
    max_height = nil, -- These can be integers or a float between 0 and 1.
    max_width = nil, -- Floats will be treated as percentage of your screen.
    border = "single", -- Border style. Can be "single", "double" or "rounded"
    mappings = {
      close = { "q", "<Esc>" },
    },
  },
  windows = { indent = 1 },
})
```

新建 dap/dap-virtual-text.lua 文件：

```lua
require("nvim-dap-virtual-text").setup {
    enabled = true,                     -- enable this plugin (the default)
    enabled_commands = true,            -- create commands DapVirtualTextEnable, DapVirtualTextDisable, DapVirtualTextToggle, (DapVirtualTextForceRefresh for refreshing when debug adapter did not notify its termination)
    highlight_changed_variables = true, -- highlight changed values with NvimDapVirtualTextChanged, else always NvimDapVirtualText
    highlight_new_as_changed = true,   -- highlight new variables in the same way as changed variables (if highlight_changed_variables)
    show_stop_reason = true,            -- show stop reason when stopped for exceptions
    commented = false,                  -- prefix virtual text with comment string
    -- experimental features:
    virt_text_pos = 'eol',              -- position of virtual text, see `:h nvim_buf_set_extmark()`
    all_frames = false,                 -- show virtual text for all stack frames not only current. Only works for debugpy on my machine.
    virt_lines = false,                 -- show virtual lines instead of virtual text (will flicker!)
    virt_text_win_col = nil             -- position the virtual text at a fixed window column (starting from the first text column) ,
                                        -- e.g. 80 to position at column 80, see `:h nvim_buf_set_extmark()`
}
```

新建 dap/config.lua 文件：

```lua
local M = {}

local function config_dapi_and_sign()
  local dap_install = require "dap-install"
  dap_install.setup {
    installation_path = vim.fn.stdpath "data" .. "/dapinstall/",
  }

  local dap_breakpoint = {
    error = {
      text = "🛑",
      texthl = "LspDiagnosticsSignError",
      linehl = "",
      numhl = "",
    },
    rejected = {
      text = "",
      texthl = "LspDiagnosticsSignHint",
      linehl = "",
      numhl = "",
    },
    stopped = {
      text = "⭐️",
      texthl = "LspDiagnosticsSignInformation",
      linehl = "DiagnosticUnderlineInfo",
      numhl = "LspDiagnosticsSignInformation",
    },
  }

  vim.fn.sign_define("DapBreakpoint", dap_breakpoint.error)
  vim.fn.sign_define("DapStopped", dap_breakpoint.stopped)
  vim.fn.sign_define("DapBreakpointRejected", dap_breakpoint.rejected)
end

local function config_dapui()
  -- dapui config
  local dap, dapui = require "dap", require "dapui"
  dap.listeners.after.event_initialized["dapui_config"] = function()
    dapui.open()
    vim.api.nvim_command("DapVirtualTextEnable")
    -- dapui.close("tray")
  end
  dap.listeners.before.event_terminated["dapui_config"] = function()
    vim.api.nvim_command("DapVirtualTextDisable")
    dapui.close()
  end
  dap.listeners.before.event_exited["dapui_config"] = function()
    vim.api.nvim_command("DapVirtualTextDisable")
    dapui.close()
  end
  -- for some debug adapter, terminate or exit events will no fire, use disconnect reuest instead
  dap.listeners.before.disconnect["dapui_config"] = function()
    vim.api.nvim_command("DapVirtualTextDisable")
    dapui.close()
  end
  -- TODO: wait dap-ui for fix temrinal layout
  -- the "30" of "30vsplit: doesn't work
  dap.defaults.fallback.terminal_win_cmd = '30vsplit new' -- this will be override by dapui
end


local function configure_debuggers()
  -- load from json file
  require('dap.ext.vscode').load_launchjs(nil, {cppdbg = {'cpp'}})
  
  -- config per launage
  -- require("user.dap.dap-cpp")
  require("user.dap.di-cpp")
  require("user.dap.dap-go")
  -- require("user.dap.di-go")
  require ("user.dap.di-python")
  -- require("user.dap.dap-cpp")
  -- require("config.dap.python").setup()
  -- require("config.dap.rust").setup()
  -- require("config.dap.go").setup()
end

function M.setup()
  config_dapi_and_sign()
  config_dapui()
  configure_debuggers() -- Debugger
end

return M
```

这里最重要的是 configure_debuggers() 函数， 针对你需要调试的语言，新建相应的文件，并添加 require 导入语句。

比如 dap/di-cpp.lua 文件：

```lua
local dap_install = require("dap-install")
dap_install.config(
  "ccppr_vsc",		-- 这个名字可以通过 :help DAPInstall.nvim 获取
{})
```

至此，基础配置完成，还需要配置快捷键， 打开keymaps.lua 文件

```lua
keymap("n", "<leader>db", "<cmd>lua require'dap'.toggle_breakpoint()<cr>", opts)
keymap("n", "<leader>dB", "<cmd>lua require'dap'.set_breakpoint(vim.fn.input '[Condition] > ')<cr>", opts)
keymap("n", "<leader>dr", "lua require'dap'.repl.open()<cr>", opts)
keymap("n", "<leader>dl", "lua require'dap'.run_last()<cr>", opts)
keymap('n', '<F10>', '<cmd>lua require"user.dap.dap-util".reload_continue()<CR>', opts)
keymap("n", "<F4>", "<cmd>lua require'dap'.terminate()<cr>", opts)
keymap("n", "<F5>", "<cmd>lua require'dap'.continue()<cr>", opts)
keymap("n", "<F6>", "<cmd>lua require'dap'.step_over()<cr>", opts)
keymap("n", "<F7>", "<cmd>lua require'dap'.step_into()<cr>", opts)
keymap("n", "<F8>", "<cmd>lua require'dap'.step_out()<cr>", opts)
keymap("n", "K", "<cmd>lua require'dapui'.eval()<cr>", opts)
```

之后可通过F5开始调试。

不过和lsp配置章节相同，即使配置至此，依然无法开始调试，因为我们还需要安装 debugger adapter，但是这个过程DapInstall.nvim已经帮我们解决了。执行 :DIInstall xxx 即可。

![image-20220411111400135](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20220411111400135.png)

## 6. 文件目录+搜索增强

至此，已经可用neovim写代码，调试代码了。但是用起来还是很别扭，因为连基本的目录导航都没有，更别说搜索功能了。现在就来解决这个问题：

```lua
  use "kyazdani42/nvim-tree.lua"     -- file explore
  -- Telescope
  use "nvim-telescope/telescope.nvim"
  use {
    "nvim-telescope/telescope-fzf-native.nvim",
    run = "make",
  }
 use "nvim-telescope/telescope-ui-select.nvim"
 use "nvim-telescope/telescope-live-grep-raw.nvim"
 use "BurntSushi/ripgrep" -- ripgrep
```

*对于telescope插件，需要安装rg和fd命令*。

建立 config/nvim-tree.lua ：

```lua
-- https://github.com/kyazdani42/nvim-tree.lua
-- 
-- <CR> or o on the root folder will cd in the above directory
-- <C-]> will cd in the directory under the cursor
-- <BS> will close current opened directory or parent
-- type a to add a file. Adding a directory requires leaving a leading / at the end of the path.
-- you can add multiple directories by doing foo/bar/baz/f and it will add foo bar and baz directories and f as a file
-- 
-- type r to rename a file
-- type <C-r> to rename a file and omit the filename on input
-- type x to add/remove file/directory to cut clipboard
-- type c to add/remove file/directory to copy clipboard
-- type y will copy name to system clipboard
-- type Y will copy relative path to system clipboard
-- type gy will copy absolute path to system clipboard
-- type p to paste from clipboard. Cut clipboard has precedence over copy (will prompt for confirmation)
-- type d to delete a file (will prompt for confirmation)
-- type D to trash a file (configured in setup())
-- type ]c to go to next git item
-- type [c to go to prev git item
-- type - to navigate up to the parent directory of the current file/directory
-- type s to open a file with default system application or a folder with default file manager (if you want to change the command used to do it see :h nvim-tree.setup under system_open)
-- if the file is a directory, <CR> will open the directory otherwise it will open the file in the buffer near the tree
-- if the file is a symlink, <CR> will follow the symlink (if the target is a file)
-- <C-v> will open the file in a vertical split
-- <C-x> will open the file in a horizontal split
-- <C-t> will open the file in a new tab
-- <Tab> will open the file as a preview (keeps the cursor in the tree)
-- I will toggle visibility of hidden folders / files
-- H will toggle visibility of dotfiles (files/folders starting with a .)
-- R will refresh the tree
-- Double left click acts like <CR>
-- Double right click acts like <C-]>
-- W will collapse the whole tree
-- S will prompt the user to enter a path and then expands the tree to match the path
-- . will enter vim command mode with the file the cursor is on
-- C-k will toggle a popup with file infos about the file under the cursor
--
-- following options are the default
-- each of these are documented in `:help nvim-tree.OPTION_NAME`
vim.g.nvim_tree_icons = {
  default = "",
  symlink = "",
  git = {
    unstaged = "",
    staged = "S",
    unmerged = "",
    renamed = "➜",
    deleted = "",
    untracked = "U",
    ignored = "◌",
  },
  folder = {
    default = "",
    open = "",
    empty = "",
    empty_open = "",
    symlink = "",
  },
}

local status_ok, nvim_tree = pcall(require, "nvim-tree")
if not status_ok then
  vim.notify("nvim-tree not found!")
  return
end

local config_status_ok, nvim_tree_config = pcall(require, "nvim-tree.config")
if not config_status_ok then
  return
end

local tree_cb = nvim_tree_config.nvim_tree_callback

nvim_tree.setup {
  disable_netrw = true,
  hijack_netrw = true,
  open_on_setup = false,
  ignore_ft_on_setup = {
    "startify",
    "dashboard",
    "alpha",
    "startup",
  },
  -- auto_close = true,
  open_on_tab = false,
  hijack_cursor = false,
  update_cwd = true,
  diagnostics = {
    enable = true,
    icons = {
      hint = "",
      info = "",
      warning = "",
      error = "",
    },
  },
  update_focused_file = {
    enable = true,
    update_cwd = false,
    ignore_list = {},
  },
  git = {
    enable = true,
    ignore = true,
    timeout = 500,
  },
  view = {
    width = 30,
    height = 30,
    hide_root_folder = false,
    side = "left",
    auto_resize = true,
    mappings = {
      custom_only = false,
      list = {
        { key = { "l", "<CR>", "o" }, action = "edit" },
        { key = "h", action ="close_node" },
        { key = "v", action ="vsplit" },
        { key = "O", action ="cd" },
      },
    },
    number = false,
    relativenumber = false,
  },
  quit_on_open = 0,
  git_hl = 1,
  disable_window_picker = 0,
  -- root_folder_modifier = ":t",
  show_icons = {
    git = 1,
    folders = 1,
    files = 1,
    folder_arrows = 1,
    tree_width = 30,
  },
  actions = {
    open_file = {
        resize_window = true    -- close half-screen usage when open a new file
    }
  }
}

-- with relative path
require"nvim-tree.events".on_file_created(function(file) vim.cmd("edit "..file.fname) end)
-- with absolute path
-- require"nvim-tree.events".on_file_created(function(file) vim.cmd("edit "..vim.fn.fnamemodify(file.fname, ":p")) end)
```

安装完成后，可通过NvimTree相关命令操作目录导航：

![image-20220411111645080](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20220411111645080.png)

当然，也可自定义快捷键。

现在在来谈谈telescope. 这也是个非常强大的插件，可以完成各种搜索，如：

文件搜索：

![image-20220411111809362](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20220411111809362.png)

文字搜索：

![image-20220411111835916](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20220411111835916.png)

项目符号搜索：

![image-20220411111925245](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20220411111925245.png)

git相关：

![image-20220411112009597](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20220411112009597.png)

还有太多。。。无法一一展示。

可通过Telescope命令补全来看支持哪些命令：

![image-20220411112100761](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20220411112100761.png)

现在来配置它，新建config/telescope.lua文件：

```lua
-- NOTE: install ripgrep for live_grep picker

-- live_grep:
-- for rp usage: reference: https://segmentfault.com/a/1190000016170184
-- -i ignore case
-- -s 大小写敏感
-- -w match word
-- -e 正则表达式匹配
-- -v 反转匹配
-- -g 通配符文件或文件夹，可以用!来取反

local status_ok, telescope = pcall(require, "telescope")
if not status_ok then
  vim.notify("telescope not found!")
  return
end

local actions = require "telescope.actions"

-- disable preview binaries
local previewers = require("telescope.previewers")
local Job = require("plenary.job")
local new_maker = function(filepath, bufnr, opts)
  filepath = vim.fn.expand(filepath)
  Job:new({
    command = "file",
    args = { "--mime-type", "-b", filepath },
    on_exit = function(j)
      local mime_type = vim.split(j:result()[1], "/")[1]
      if mime_type == "text" then
        previewers.buffer_previewer_maker(filepath, bufnr, opts)
      else
        -- maybe we want to write something to the buffer here
        vim.schedule(function()
          vim.api.nvim_buf_set_lines(bufnr, 0, -1, false, { "BINARY" })
        end)
      end
    end
  }):sync()
end

telescope.setup {
  defaults = {
    buffer_previewer_maker = new_maker,

    prompt_prefix = " ",
    selection_caret = " ",
    path_display = {
      shorten= {
        -- e.g. for a path like
        --   `alpha/beta/gamma/delta.txt`
        -- setting `path_display.shorten = { len = 1, exclude = {1, -1} }`
        -- will give a path like:
        --   `alpha/b/g/delta.txt`
        len = 3, exclude = {1, -1}
      },
    },

    mappings = {
      i = {
        ["<C-n>"] = actions.cycle_history_next,
        ["<C-p>"] = actions.cycle_history_prev,

        ["<C-j>"] = actions.move_selection_next,
        ["<C-k>"] = actions.move_selection_previous,

        ["<C-c>"] = actions.close,

        ["<Down>"] = actions.move_selection_next,
        ["<Up>"] = actions.move_selection_previous,

        ["<CR>"] = actions.select_default,
        ["<C-x>"] = actions.select_horizontal,
        ["<C-v>"] = actions.select_vertical,
        ["<C-t>"] = actions.select_tab,

        ["<C-u>"] = actions.preview_scrolling_up,
        ["<C-d>"] = actions.preview_scrolling_down,

        ["<PageUp>"] = actions.results_scrolling_up,
        ["<PageDown>"] = actions.results_scrolling_down,

        ["<Tab>"] = actions.toggle_selection + actions.move_selection_worse,
        ["<S-Tab>"] = actions.toggle_selection + actions.move_selection_better,
        ["<C-q>"] = actions.send_to_qflist + actions.open_qflist,
        ["<M-q>"] = actions.send_selected_to_qflist + actions.open_qflist,
        ["<C-l>"] = actions.complete_tag,
        ["<C-_>"] = actions.which_key, -- keys from pressing <C-/>
      },

      n = {
        ["<esc>"] = actions.close,
        ["<CR>"] = actions.select_default,
        ["<C-x>"] = actions.select_horizontal,
        ["<C-v>"] = actions.select_vertical,
        ["<C-t>"] = actions.select_tab,

        ["<Tab>"] = actions.toggle_selection + actions.move_selection_worse,
        ["<S-Tab>"] = actions.toggle_selection + actions.move_selection_better,
        ["<C-q>"] = actions.send_to_qflist + actions.open_qflist,
        ["<M-q>"] = actions.send_selected_to_qflist + actions.open_qflist,

        ["j"] = actions.move_selection_next,
        ["k"] = actions.move_selection_previous,
        ["H"] = actions.move_to_top,
        ["M"] = actions.move_to_middle,
        ["L"] = actions.move_to_bottom,

        ["<Down>"] = actions.move_selection_next,
        ["<Up>"] = actions.move_selection_previous,
        ["gg"] = actions.move_to_top,
        ["G"] = actions.move_to_bottom,

        ["<C-u>"] = actions.preview_scrolling_up,
        ["<C-d>"] = actions.preview_scrolling_down,

        ["<PageUp>"] = actions.results_scrolling_up,
        ["<PageDown>"] = actions.results_scrolling_down,

        ["?"] = actions.which_key,
      },
    },
  },
  pickers = {
    find_files = {
      theme = "dropdown",
      previewer = false,
      -- find_command = { "find", "-type", "f" },
      find_command = {"fd"},
    },

    -- Default configuration for builtin pickers goes here:
    -- picker_name = {
    --   picker_config_key = value,
    --   ...
    -- }
    -- Now the picker_config_key will be applied every time you call this
    -- builtin picker
  },
  extensions = {
    -- Your extension configuration goes here:
    -- extension_name = {
    --   extension_config_key = value,
    -- }

    -- fzf syntax
    -- Token	Match type	Description
    -- sbtrkt	fuzzy-match	Items that match sbtrkt
    -- 'wild'	exact-match (quoted)	Items that include wild
    -- ^music	prefix-exact-match	Items that start with music
    -- .mp3$	suffix-exact-match	Items that end with .mp3
    -- !fire	inverse-exact-match	Items that do not include fire
    -- !^music	inverse-prefix-exact-match	Items that do not start with music
    -- !.mp3$	inverse-suffix-exact-match	Items that do not end with .mp3
    fzf = {
      fuzzy = true,                    -- false will only do exact matching
      override_generic_sorter = true,  -- override the generic sorter
      override_file_sorter = true,     -- override the file sorter
      case_mode = "smart_case",        -- or "ignore_case" or "respect_case"
      -- the default case_mode is "smart_case"
    },
    ["ui-select"] = {
      require("telescope.themes").get_dropdown {
        -- even more opts
      }
    },
  },
}

-- telescope.load_extension("frecency")
telescope.load_extension('fzf')
telescope.load_extension("ui-select")
telescope.load_extension('dap')
-- load project extension. see project.lua file
```

然后可通过绑定一定快捷键来操作各种窗口。

## 7. Git相关

**推荐直接使用lazygit做git项目管理，neovim中可以做一些hunk相关操作和file diff管理**

```lua
  -- Git
  use "lewis6991/gitsigns.nvim"
  use "tanvirtin/vgit.nvim" 
```

diff hunk:

![image-20220411112650228](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20220411112650228.png)

diff file：

![image-20220411112617303](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20220411112617303.png)

file history:

![image-20220411112724466](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20220411112724466.png)

xxx

这里只说下Gitsign的配置，更多请参考github, 新建 config/gitsigns.lua 文件

```lua
local status_ok, gitsigns = pcall(require, "gitsigns")
if not status_ok then
  vim.notify("gitsigns not found!")
  return
end

gitsigns.setup {
  signs = {
    add          = {hl = 'GitSignsAdd'   , text = '│', numhl='GitSignsAddNr'   , linehl='GitSignsAddLn'},
    change       = {hl = 'GitSignsChange', text = '│', numhl='GitSignsChangeNr', linehl='GitSignsChangeLn'},
    delete       = {hl = 'GitSignsDelete', text = '_', numhl='GitSignsDeleteNr', linehl='GitSignsDeleteLn'},
    topdelete    = {hl = 'GitSignsDelete', text = '‾', numhl='GitSignsDeleteNr', linehl='GitSignsDeleteLn'},
    changedelete = {hl = 'GitSignsChange', text = '~', numhl='GitSignsChangeNr', linehl='GitSignsChangeLn'},
  },
  signcolumn = true, -- Toggle with `:Gitsigns toggle_signs`
  numhl = false, -- Toggle with `:Gitsigns toggle_numhl`
  linehl = false, -- Toggle with `:Gitsigns toggle_linehl`
  word_diff = false, -- Toggle with `:Gitsigns toggle_word_diff`
  watch_gitdir = {
    interval = 1000, follow_files = true,
  },
  attach_to_untracked = true,
  current_line_blame = true, -- Toggle with `:Gitsigns toggle_current_line_blame`
  current_line_blame_opts = {
    virt_text = true,
    virt_text_pos = "eol", -- 'eol' | 'overlay' | 'right_align'
    delay = 100,
    ignore_whitespace = false,
  },
  current_line_blame_formatter_opts = {
    relative_time = false,
  },
  sign_priority = 6,
  update_debounce = 100,
  status_formatter = nil, -- Use default
  max_file_length = 40000,
  preview_config = {
    -- Options passed to nvim_open_win
    border = "single",
    style = "minimal",
    relative = "cursor",
    row = 0,
    col = 1,
  },
  yadm = {
    enable = false,
  },
  -- keymapping
  on_attach = function(bufnr)
    local function map(mode, lhs, rhs, opts)
        opts = vim.tbl_extend('force', {noremap = true, silent = true}, opts or {})
        vim.api.nvim_buf_set_keymap(bufnr, mode, lhs, rhs, opts)
    end

    -- Navigation
    map('n', '<leader>hj', ':Gitsigns next_hunk<CR>')
    map('n', '<leader>hk',':Gitsigns prev_hunk<CR>')

    -- Actions
    map('n', '<leader>hs', ':Gitsigns stage_hunk<CR>')
    map('n', '<leader>hr', ':Gitsigns reset_hunk<CR>')
    map('n', '<leader>hu', '<cmd>Gitsigns undo_stage_hunk<CR>')
    map('n', '<leader>hS', '<cmd>Gitsigns stage_buffer<CR>')
    map('n', '<leader>hR', '<cmd>Gitsigns reset_buffer<CR>')
    map('n', '<leader>hp', '<cmd>Gitsigns preview_hunk<CR>')
    map('n', '<leader>hb', '<cmd>lua require"gitsigns".blame_line{full=true}<CR>')
    map('n', '<leader>tb', '<cmd>Gitsigns toggle_current_line_blame<CR>')
    map('n', '<leader>hd', '<cmd>Gitsigns diffthis<CR>')
    map('n', '<leader>hD', '<cmd>lua require"gitsigns".diffthis("~")<CR>')
    map('n', '<leader>td', '<cmd>Gitsigns toggle_deleted<CR>')
    --
    -- Text object
    map('o', 'ih', ':<C-U>Gitsigns select_hunk<CR>')
    map('x', 'ih', ':<C-U>Gitsigns select_hunk<CR>')
  end
}
```

## 8. 其它

还有很多有用的插件，这里不一一列举如何配置，提一些个人觉得十分有用的：

- windwp/nvim-autopairs， 自动补全括号
- terrortylor/nvim-comment， 代码注释
- rmagatti/auto-session，保存编辑session，下次通过nvim命令直接恢复至上次退出nvim时的布局和打开过的文件
- haringsrob/nvim_context_vt， 可以在for、if、function等语句结束时添加一些虚拟文件，辅助看代码挺有用的。
- tpope/vim-surround， 更改surround的字符，算是经典的老插件了
- akinsho/toggleterm.nvim， neovim终端增强
- phaazon/hop.nvim， 类似，vim中的easymotion，不过更强大
- stevearc/aerial.nvim， 代码大纲导航
- folke/trouble.nvim， 非常强大的视图增强插件，个人用它看reference, 看todo comment等。
- sindrets/winshift.nvim，移动neovim中的window
- ldelossa/litee.nvim一系列插件，对neovim中缺失的ide功能进行补全，比如查看call graph
- cdelledonne/vim-cmake， 在nvim中执行cmake命令
- AckslD/nvim-neoclip.lua， 拷贝历史
- vim-test/vim-test和rcarriga/vim-ultest, 单元测试目录，调试等，支持python、go等常用语言，但是不支持c++的google test（很可惜，不过目前c++有人也在提pr了，希望未来也能支持c++）

更多可参考github。

