# SHELL

## 权限

首先是文件的权限，每个文件都有十个权限位，分别是：

- 第1位：文件的类型，分为d, -, l
- 第1组(2-4)：文件所有者的权限
- 第2组：文件所有组的权限
- 第3组：其他用户的权限

## 流

重定向是`>`和`<`

注意管道`|`的用法，他可以把左侧的输出转入右侧的输入

tee是一个很重要的指令

## su

使用root的终端时，注意权限问题



# Script

我们想用shell执行一系列命令，且他们之间可能有诸如条件选择和循环等复杂关系，这时一个个的去输入命令显得很蠢。为了自动化命令的执行，我们有了 `Shell Script`

## Shell Script

### assign

最简单的变量赋值

```bash
foo=bar
echo "$foo"
# bar
```

注意赋值时中间不要间隔空格

### function & args

```bash
mcd(){
	mkdir -p "$1"
	cd "$1"
}
```

在上面的函数定义中， `$1` 是这个脚本的参数，具体含义见下：

- `$0` - Name of the script
- `$1` to `$9` - Arguments to the script. `$1` is the first argument and so on.
- `$@` - All the arguments
- `$#` - Number of arguments
- `$?` - Return code of the previous command
- `$$` - Process identification number (PID) for the current script
- `!!` - Entire last command, including arguments. A common pattern is to execute a command only for it to fail due to missing permissions; you can quickly re-execute the command with sudo by doing `sudo !!`
- `$_` - Last argument from the last command. If you are in an interactive shell, you can also quickly get this value by typing `Esc` followed by `.` or `Alt+.`

### CMD with variables

- *command substitution：*`$(CMD)` - 执行CMD，并且以他的结果作为变量
- *process substitution：* `<(CMD)` - 执行CMD，将他的结果储存在一个临时文件中，返回该文件的文件名。这样做的是因为一些命令只接受文件名作为输入

### 通配符

以下是一些有用的通配符：

- `*`
- `?`
- `{}`
- `..`

## Shell Tools

### tldr

`tldr`是很好的 `man` 的简单替代品，提供更简洁的命令说明以及示例

### find

这里直接列出课程中给出的一些关于 `find` 的妙用

```bash
# Find all directories named src
find . -name src -type d
# Find all python files that have a folder named test in their path
find . -path '*/test/*.py' -type f
# Find all files modified in the last day
find . -mtime -1
# Find all zip files with size in range 500k to 10M
find . -size +500k -size -10M -name '*.tar.gz'
# Delete all files with .tmp extension
find . -name '*.tmp' -exec rm {} \\;
# Find all PNG files and convert them to JPG
find . -name '*.png' -exec convert {} {}.jpg \\;
```

### grep & rg

`rg` 是对 `grep -R` 的替代。对于单文件的内容搜索我们已经有很多实践，但是对于目录的递归搜索也就是 `grep -R` ，它的功能不够强大，所以用 `rg` 可以作为很好的补充

```bash
# Find all python files where I used the requests library
rg -t py 'import requests'
# Find all files (including hidden files) without a shebang line
rg -u --files-without-match "^#\\!"
# Find all matches of foo and print the following 5 lines
rg foo -A 5
# Print statistics of matches (# of matched lines and files )
rg --stats PATTERN
```

### autojump

`autojump` 可以做到快速访问一些目录。它添加了命令 `j` ，用它加上一些筛选条件即可快速跳转

# Vim

## modal

Vim is a multi-modal editor, it has `normal` mode at beginning, and u can enter other modes by pressing different keys in normal mode:

- `i` - insert
- `r` - replace
- `v` - visual, including plain, line and block `S-V` : line `C-V` : block
- `:` - command line

## Command-line

- `:q` quit (close window)

- `:w` save (“write”)

- `:wq` save and quit

- `:e {name of file}` open file for editing

- `:ls` show open buffers

- ```
  :help {topic}
  ```

   open help

  - `:help :w` opens help for the `:w` command
  - `:help w` opens help for the `w` movement

## Movement

Those movements are also called `nouns`, they can be used with `verbs`

- Basic movement: `hjkl` (left, down, up, right)

- Words: `w` (next word), `b` (beginning of word), `e` (end of word)

- Lines: `0` (beginning of line), `^` (first non-blank character), `$` (end of line)

- Screen: `H` (top of screen), `M` (middle of screen), `L` (bottom of screen)

- Scroll: `Ctrl-u` (up), `Ctrl-d` (down)

- File: `gg` (beginning of file), `G` (end of file)

- Line numbers: `:{number}<CR>` or `{number}G` (line {number})

- Misc: `%` (corresponding item)

- Find: 

  ```
  f{character}
  ```

  , 

  ```
  t{character}
  ```

  , 

  ```
  F{character}
  ```

  , 

  ```
  T{character}
  ```

  - find/to forward/backward {character} on the current line
  - `,` / `;` for navigating matches

- Search: `/{regex}`, `n` / `N` for navigating matches

## Edit

Edits are also called `verbs` , they can be used with `nouns`

- `i` enter Insert mode: but for manipulating/deleting text, want to use something more than backspace

- `o` / `O` insert line below / above

- ```
  d{motion}
  ```

   delete {motion}

  - e.g. `dw` is delete word, `d$` is delete to end of line, `d0` is delete to beginning of line

- ```
  c{motion}
  ```

   change {motion}

  - e.g. `cw` is change word
  - like `d{motion}` followed by `i`

- `x` delete character (equal do `dl`)

- `s` substitute character (equal to `cl`)

- Visual mode + manipulation

  - select text, `d` to delete it or `c` to change it

- `u` to undo, `<C-r>` to redo

- `y` to copy / “yank” (some other commands like `d` also copy)

- `p` to paste

- Lots more to learn: e.g. `~` flips the case of a character

## Count

Combine `nouns` or `verbs` with counts, like these:

- `3w` move 3 words forward
- `5j` move 5 lines down
- `7dw` delete 7 words

## Modifier

Modifiers are used to change the meaning of `nouns`

- `ci(` change the contents inside the current pair of parentheses
- `ci[` change the contents inside the current pair of square brackets
- `da'` delete a single-quoted string, including the surrounding single quotes



# Data

Data wrangling is to take data in one format and turn it into a different format, it’s quite a complicated thing, so in this note, I only record some useful tools and how to use them

## Tools

```
|
```

pipe operator, it is used to connect the output of one command to the input of another.

```
grep
less
```

It is used to show data in a **pager, n**ot giving less info

```
sed
```

It means “stream editor”. It gives basic commands to modify the file, the most common one `s`  can be use in a form like: `s/REGEX/SUBSTITUTION/`, it can do **search&replace**

```bash
ssh myserver journalctl
 | grep sshd
 | grep "Disconnected from"
 | sed 's/.*Disconnected from //'
sort
```

sort the input, but not only sort, it can do more with the args, like sort by specific columns

```
uniq -c
```

collapse the same line into a single

```
head` & `tail
```

only show head/tail lines

```
awk
```

it’s a powerful programming language and good at processing text streams, maybe it can be useful, but I think it’s not worth learning because python can do these things, too.

```
bc
```

A calculator in shell, it can read from STDIN.

## Regex

- `.` means “any single character” except newline
- `*` zero or more of the preceding match
- `+` one or more of the preceding match
- `[abc]` any one character of `a`, `b`, and `c`
- `(RX1|RX2)` either something that matches `RX1` or `RX2`
- `^` the start of the line
- `$` the end of the line

It’s really a hard thing to write a right regex, so in real situation, it’s better to use some regex that already have.



# CommandLine Environment

## Job Control

### Signals

### **Pausing and backgrounding processes**

We can continue the paused job in the foreground or in the background using `fg` or `bg`, respectively.

`jobs` command lists the unfinished jobs associated with the current terminal session

`&` suffix in a command will run the command in the background

```
bg` will make background processes to be children processes of the terminal, so when terminal is closed, all background processes will die. To prevent this, `nohup` is a good way, or using `tmux
```

## Terminal Multiplexers

Manage different workspace

session → window → pane

**Sessions** - a session is an independent workspace with one or more windows

- `tmux` starts a new session.
- `tmux new -s NAME` starts it with that name.
- `tmux ls` lists the current sessions
- Within `tmux` typing `<C-b> d` detaches the current session
- `tmux a` attaches the last session. You can use `t` flag to specify which

**Windows** - Equivalent to tabs in editors or browsers, they are visually separate parts of the same session

- `<C-b> c` Creates a new window. To close it you can just terminate the shells doing `<C-d>`
- `<C-b> N` Go to the *N* th window. Note they are numbered
- `<C-b> p` Goes to the previous window
- `<C-b> n` Goes to the next window
- `<C-b> ,` Rename the current window
- `<C-b> w` List current windows

**Panes** - Like vim splits, panes let you have multiple shells in the same visual display.

- `<C-b> "` Split the current pane horizontally
- `<C-b> %` Split the current pane vertically
- `<C-b> <direction>` Move to the pane in the specified *direction*. Direction here means arrow keys.
- `<C-b> z` Toggle zoom for the current pane
- `<C-b> [` Start scrollback. You can then press `<space>` to start a selection and `<enter>` to copy that selection.
- `<C-b> <space>` Cycle through pane arrangements.

## Aliases

Make aliases to some commands you often use or those commands that you always type wrong in the shell startup files like `.bashrc`

## Dotfiles

Using git and some auto-tools to make dotfiles portable and syncable.

## Remote Machines

Use `ssh` to connect to the remote machines.

SSH keys can help us login without enter password:

```bash
ssh-keygen -o -a 100 -t ed25519 -f ~/.ssh/id_ed25519
ssh-copy-id -i .ssh/id_ed25519 {remote_machine_name}
```

### Port Forwarding

### SSH config

`~/.ssh/config` file saves the configuration of your ssh