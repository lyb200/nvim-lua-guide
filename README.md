# Getting started using Lua in Neovim

## Introduction

The integration of Lua as a first-class language inside Neovim is shaping up to be one of its killer features. However, the amount of teaching material for learning how to write plugins in Lua is not as large as what you would find for writing them in Vimscript. This is an attempt at providing some basic information to get people started.

This guide assumes you are using the latest [nightly build](https://github.com/neovim/neovim/releases/tag/nightly) of Neovim. Since version 0.5 of Neovim is a development version, keep in mind that some APIs that are being actively worked on are not quite stable and might change before release.

### Learning Lua

If you are not already familiar with the language, there are plenty of resources to get started:

- The [Learn X in Y minutes page about Lua](https://learnxinyminutes.com/docs/lua/) should give you a quick overview of the basics
- If videos are more to your liking, Derek Banas has a [1-hour tutorial on the language](https://www.youtube.com/watch?v=iMacxZQMPXs)
- The [lua-users wiki](http://lua-users.org/wiki/LuaDirectory) is full of useful information on all kinds of Lua-related topics
- The [official reference manual for Lua](https://www.lua.org/manual/5.1/) should give you the most comprehensive tour of the language

It should also be noted that Lua is a very clean and simple language. It is easy to learn, especially if you have experience with similar scripting languages like JavaScript. You may already know more Lua than you realise!

Note: the version of Lua that Neovim embeds is LuaJIT 2.1.0, which maintains compatibility with Lua 5.1 (with a few 5.2 extensions)

### Existing tutorials for writing Lua in Neovim

A few tutorials have already been written to help people write plugins in Lua. Some of them helped quite a bit when writing this guide. Many thanks to their authors.

- [teukka.tech - From init.vim to init.lua](https://teukka.tech/luanvim.html)
- [2n.pl - How to write neovim plugins in Lua](https://www.2n.pl/blog/how-to-write-neovim-plugins-in-lua.md)
- [2n.pl - How to make UI for neovim plugins in Lua](https://www.2n.pl/blog/how-to-make-ui-for-neovim-plugins-in-lua)
- [ms-jpq - Neovim Async Tutorial](https://ms-jpq.github.io/neovim-async-tutorial/)

## Where to put Lua files

Lua files are typically found inside a `lua/` folder in your `runtimepath` (for most users, this will mean `~/.config/nvim/lua` on *nix systems and `~/AppData/Local/nvim/lua` on Windows). The `package.path` and `package.cpath` globals are automatically adjusted to include Lua files in this folder. This means you can `require()` these files as Lua modules.

Let's take the following folder structure as an example:

```
📂 ~/.config/nvim
├── 📁 after
├── 📁 ftplugin
├── 📂 lua
│  ├── 🌑 myluamodule.lua
│  └── 📂 other_modules
│     ├── 🌑 anothermodule.lua
│     └── 🌑 init.lua
├── 📁 pack
├── 📁 plugin
├── 📁 syntax
└── 🇻 init.vim
```

The following Lua code will load `myluamodule.lua`:

```lua
require('myluamodule')
```

Notice the absence of a `.lua` extension.

Similarly, loading `other_modules/anothermodule.lua` is done like so:

```lua
require('other_modules.anothermodule')
-- or
require('other_modules/anothermodule')
```

Path separators are denoted by either a dot `.` or a slash `/`.

A folder containing an `init.lua` file can be required directly, without have to specify the name of the file.

```lua
require('other_modules') -- loads other_modules/init.lua
```

For more information: `:help lua-require`

#### Caveats

Unlike .vim files, .lua files are not automatically sourced from directories in your `runtimepath`. Instead, you have to source/require them from Vimscript. There are plans to add the option to load an `init.lua` file as an alternative to `init.vim`:

- [Issue #7895](https://github.com/neovim/neovim/issues/7895)
- [Corresponding pull request](https://github.com/neovim/neovim/pull/12235)

#### Tips

Several Lua plugins might have identical filenames in their `lua/` folder. This could lead to namespace clashes.

If two different plugins have a `lua/main.lua` file, then doing `require('main')` is ambiguous: which file do we want to source?

It might be a good idea to namespace your config or your plugin with a top-level folder, like so: `lua/plugin_name/main.lua`

## Using Lua from Vimscript

### :lua

This command executes a chunk of Lua code.

```vim
:lua require('myluamodule')
```

Multi-line scripts are possible using heredoc syntax:

```vim
echo "Here's a bigger chunk of Lua code"

lua << EOF
local mod = require('mymodule')
local tbl = {1, 2, 3}

for k, v in ipairs(tbl) do
    mod.method(v)
end

print(tbl)
EOF
```

See also:

- `:help :lua`
- `:help :lua-heredoc`

#### Caveats

You don't get correct syntax highlighting when writing Lua in a .vim file. It might be more convenient to use the `:lua` command as an entry point for requiring external Lua files.

### :luado

This command executes a chunk of Lua code that acts on a range of lines in the current buffer. If no range is specified, the whole buffer is used instead. Whatever string is `return`ed from the chunk is used to determine what each line should be replaced with.

The following command would replace every line in the current buffer with the text `hello world`:

```vim
:luado return 'hello world'
```

Two implicit `line` and `linenr` variables are also provided. `line` is the text of the line being iterated upon whereas `linenr` is its number. The following command would make every line whose number is divisible by 2 uppercase:

```vim
:luado if linenr % 2 == 0 then return line:upper() end
```

See also:

- `:help :luado`

### :luafile

This command sources a Lua file.

```vim
:luafile ~/foo/bar/baz/myluafile.lua
```

It is analogous to the `:source` command for .vim files or the built-in `dofile()` function in Lua.

See also:

- `:help :luafile`

### luaeval()

This built-in Vimscript function evaluates a Lua expression string and returns its value. Lua data types are automatically converted to Vimscript types (and vice versa).

```vim
" You can store the result in a variable
let variable = luaeval('1 + 1')
echo variable
" 2
let concat = luaeval('"Lua".." is ".."awesome"')
echo concat
" 'Lua is awesome'

" List-like tables are converted to Vim lists
let list = luaeval('{1, 2, 3, 4}')
echo list[0]
" 1
echo list[1]
" 2
" Note that unlike Lua tables, Vim lists are 0-indexed

" Dict-like tables are converted to Vim dictionaries
let dict = luaeval('{foo = "bar", baz = "qux"}')
echo dict.foo
" 'bar'

" Same thing for booleans and nil
echo luaeval('true')
" v:true
echo luaeval('nil')
" v:null

" You can create Vimscript aliases for Lua functions
let LuaMathPow = luaeval('math.pow')
echo LuaMathPow(2, 2)
" 4
let LuaModuleFunction = luaeval('require("mymodule").myfunction')
call LuaModuleFunction()
```

`luaeval()` takes an optional second argument that allows you to pass data to the expression. You can then access that data from Lua using the magic global `_A`:

```vim
echo luaeval('_A[1] + _A[2]', [1, 1])
" 2

echo luaeval('string.format("Lua is %s", _A)', 'awesome')
" 'Lua is awesome'
```

See also:
- `:help luaeval()`

### v:lua

This global Vim variable allows you to call global Lua functions directly from Vimscript. Again, Vim data types are converted to Lua types and vice versa.

```vim
call v:lua.print('Hello from Lua!')
" 'Hello from Lua!'

let scream = v:lua.string.rep('A', 10)
echo scream
" 'AAAAAAAAAA'

" Requiring modules works
call v:lua.require('mymodule').myfunction()

" How about a nice statusline?
lua << EOF
function _G.statusline()
    local filepath = '%f'
    local align_section = '%='
    local percentage_through_file = '%p%%'
    return string.format(
        '%s%s%s',
        filepath,
        align_section,
        percentage_through_file
    )
end
EOF

set statusline=%!v:lua.statusline()

" Also works in expression mappings
lua << EOF
function _G.check_back_space()
    local col = vim.fn.col('.') - 1
    if col == 0 or vim.fn.getline('.'):sub(col, col):match('%s') then
        return true
    else
        return false
    end
end
EOF

inoremap <silent> <expr> <Tab>
    \ pumvisible() ? '\<C-n>' :
    \ v:lua.check_back_space() ? '\<Tab>' :
    \ completion#trigger_completion()
```

See also:
- `:help v:lua`
- `:help v:lua-call`

#### Caveats

This variable can only be used to call functions. The following will always throw an error:

```vim
" Aliasing functions doesn't work
let LuaPrint = v:lua.print

" Accessing dictionaries doesn't work
echo v:lua.some_global_dict['key']

" Using a function as a value doesn't work
echo map([1, 2, 3], v:lua.global_callback)
```

## The vim namespace

Neovim exposes a global `vim` variable which serves as an entry point to interact with its APIs from Lua. It provides users with an extended "standard library" of functions as well as various sub-modules.

Some notable functions and modules include:

- `vim.inspect`: pretty-print Lua objects (useful for inspecting tables)
- `vim.regex`: use Vim regexes from Lua
- `vim.api`: module that exposes API functions (the same API used by remote plugins)
- `vim.loop`: module that exposes the functionality of Neovim's event-loop (using LibUV)
- `vim.lsp`: module that controls the built-in LSP client
- `vim.treesitter`: module that exposes the functionality of the tree-sitter library

This list is by no means comprehensive. If you wish to know more about what's made available by the `vim` variable, `:help lua-stdlib` and `:help lua-vim` are the way to go. Alternatively, you can do `:lua print(vim.inspect(vim))` to get a list of every module.

#### Tips

Writing `print(vim.inspect(x))` every time you want to inspect the contents of an object can get pretty tedious. It might be worthwhile to have a global wrapper function somewhere in your configuration:

```lua
function _G.dump(...)
    local objects = vim.tbl_map(vim.inspect, {...})
    print(unpack(objects))
end
```

You can then inspect the contents of an object very quickly in your code or from the command-line:

```lua
dump({1, 2, 3})
```

```vim
:lua dump(vim.loop)
```


Additionally, you may find that built-in Lua functions (such as `math.max()` or `string.rep()`) are sometimes lacking compared to what you would find in other languages (for example `os.clock()` only returns a value in seconds, not milliseconds). Be sure to look at the Neovim stdlib (and `vim.fn`, more on that later), it probably has what you're looking for.

## Using Vimscript from Lua

### vim.api.nvim_eval()

This function evaluates a Vimscript expression string and returns its value. Vimscript data types are automatically converted to Lua types (and vice versa).

It is the Lua equivalent of the `luaeval()` function in Vimscript

```lua
-- Data types are converted correctly
print(vim.api.nvim_eval('1 + 1')) -- 2
print(vim.inspect(vim.api.nvim_eval('[1, 2, 3]'))) -- { 1, 2, 3 }
print(vim.inspect(vim.api.nvim_eval('{"foo": "bar", "baz": "qux"}'))) -- { baz = "qux", foo = "bar" }
print(vim.api.nvim_eval('v:true')) -- true
print(vim.api.nvim_eval('v:null')) -- nil
```

#### Caveats

Unlike `luaeval()`, `vim.api.nvim_eval()` does not provide an implicit `_A` variable to pass data to the expression.

### vim.api.nvim_exec()

This function evaluates a chunk of Vimscript code. It takes in a string containing the source code to execute and a boolean to determine whether the output of the code should be returned by the function (you can then store the output in a variable, for example).

```lua
local result = vim.api.nvim_exec(
[[
let text = 'hello world'
echo text

function! MyFunction()
    " do something inside this function
endfunction

call MyFunction()
]],
true)

print(result) -- 'hello world'
```

### vim.api.nvim_command()

This function executes an ex command. It take in a string containing the command to execute.

```lua
vim.api.nvim_command('new')
vim.api.nvim_command('wincmd H')
vim.api.nvim_command('set nonumber')
vim.api.nvim_command('%s/foo/bar/g')
```

Note: `vim.cmd` is a shorter alias to this function (currently not documented in `:help`)

```lua
vim.cmd('buffers')
```

#### Tips

Since you have to pass strings to these functions, you often end up having to escape backslashes:

```lua
vim.cmd('%s/\\Vfoo/bar/g')
```

Literal strings are easier to use as they do not require escaping characters:

```lua
vim.cmd([[%s/\Vfoo/bar/g]])
```

## Managing vim options

### Using api functions

<!-- vim.api.nvim_set_option() -->
<!-- vim.api.nvim_get_option() -->
<!-- vim.api.nvim_buf_set_option() -->
<!-- vim.api.nvim_buf_get_option() -->
<!-- vim.api.nvim_win_set_option() -->
<!-- vim.api.nvim_win_get_option() -->

### Using meta-accessors

<!-- vim.o.{option} -->
<!-- vim.bo.{option} -->
<!-- vim.wo.{option} -->

## Managing vim internal variables

### Using api functions

<!-- vim.api.nvim_set_var() -->
<!-- vim.api.nvim_get_var() -->
<!-- vim.api.nvim_del_var() -->
<!-- vim.api.nvim_buf_set_var() -->
<!-- vim.api.nvim_buf_get_var() -->
<!-- vim.api.nvim_buf_del_var() -->
<!-- vim.api.nvim_win_set_var() -->
<!-- vim.api.nvim_win_get_var() -->
<!-- vim.api.nvim_win_del_var() -->
<!-- vim.api.nvim_tabpage_set_var() -->
<!-- vim.api.nvim_tabpage_get_var() -->
<!-- vim.api.nvim_tabpage_del_var() -->
<!-- vim.api.nvim_set_vvar() -->
<!-- vim.api.nvim_get_vvar() -->

### Using meta-accessors

<!-- vim.g.{name} -->
<!-- vim.b.{name} -->
<!-- vim.w.{name} -->
<!-- vim.t.{name} -->
<!-- vim.v.{name} -->

## Calling Vimscript functions

### vim.api.nvim_call_function()

### vim.call()

### vim.fn.{function}()

## Defining mappings

<!-- nvim_set_keymap() -->
<!-- nvim_get_keymap() -->
<!-- nvim_del_keymap() -->
<!-- nvim_buf_set_keymap() -->
<!-- nvim_buf_get_keymap() -->
<!-- nvim_buf_del_keymap() -->

## Defining user commands

<!-- https://github.com/neovim/neovim/pull/11613 -->

## Defining autocommands

<!-- TODO: Mention wrapper + pending PR -->

## Defining highlights

<!-- mention colorbuddy.nvim -->

## Making your code more robust

### vim.validate()

### Unit tests

## Miscellaneous

### vim.loop

<!-- TODO: Mention libuv docs + luvit api -->
<!-- https://teukka.tech/vimloop.html -->

### vim.lsp

### vim.treesitter

<!-- TODO: add interesting projects (transpilers) -->
<!-- https://github.com/svermeulen/nvim-moonmaker -->
<!-- https://github.com/Olical/aniseed -->
<!-- https://github.com/Olical/conjure -->
<!-- https://github.com/TypeScriptToLua/TypeScriptToLua -->
<!-- https://github.com/teal-language/tl -->
<!-- https://haxe.org/ -->
<!-- https://github.com/SwadicalRag/wasm2lua -->
<!-- https://github.com/hengestone/lua-languages -->
