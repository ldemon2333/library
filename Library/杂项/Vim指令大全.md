`i` lets you *i*nsert text before the cursor
 
`a` lets you *a*ppend text after the cursor

`I` lets you *i*nsert text at the beginning of a line

`A` lets you *a*ppend text at the end of a line

`o` lets you *o*pen a new line below the current line

`O` lets you *o*pen a new line above the current line

`gi` let you go back to the last place where you made a change

> `gi` in VSCodeVim behaves differently than in Vim. Where in Vim you go back to the last place you left Insert mode, in VSCodeVim you get into insert mode where you did you last change.

> Notice how often `g` is used as a modifier of other commands. When you see `g` before a common command you can expect that the command will do something similar to the original command: For example, `gi` lets you go to the last place you left Insert mode (`i`), `ge` does the reverse of `e`, etc...

`CTRL-H` lets you remove the last character you typed (mnemonic _h_ which in _hjkl_ brings the cursor one space to the left)

`CTRL-W` lets you remove the last word you typed (mnemonic *w*ord)

`CTRL-U` lets you remove the last line you typed (mnemonic *u*ndo this line)

> Remember that you can navigate this file using hjkl:
>
> ```
>            ↑
>      ← h j k l →
>          ↓
> ```
>
> or {} or CTRL-U, CTRL-D

`u` to undo and you last change will be reverted

`<CTRL-R>` to redo

`ggdG` to delete the whole document

`d` for delete

`f` for find

`c` for change

`t` to until

`dd` lets you delete a complete line of text

`D` lets you remove everything from the cursor until the end of the line (it is equivalent to `d$`)

> Mini-refresher: The d command
>
> - d{motion} - delete text covered by motion
>
>   - d2w => deletes two words
>   - dt; => delete until ;
>   - d/hello => delete until hello
>
> - dd - delete line
> - D - delete from cursor until the end of the line

`.` you can repeat a complete change that can be composed of as many keystrokes as you can imagine

`cc` changes a complete line

`C` changes from the cursor until the end on the line

```
            |- `a` means around
            |- `i` means inner
           /
          /
         /
        {a|i}{text-object}
                  /
                 /
                | w - word
                | s - sentence
                | p - paragraph
                | " - quotes
```

> if you have some trouble getting to the right sentence below try using `gj` instead of `j`, when you prepend `g` before `j` and `k` you can navigate wrapped lines (and not just lines).

`y` (yank): Copy in Vim jargon

`p` (put): Paste in Vim jargon

`g~` (switch case): Changes letters from lowercase to uppercase and back. Alternatively, use `gu` to make something lowercase and `gU` to make something uppercase

`>` (shift right): Adds indentation

`<` (shift left): Removes indentation

`=` (format code): Formats code


`x` is equivalent to `dl` and deletes the character under the cursor

`X` is equivalent to `dh` and deletes the character before the cursor

`s` is equivalent to `ch`, deletes the character under the cursor and puts you into Insert mode

`r` allows you to replace one single character for another. Very handy to fix typos.

`~` to switch case for a single character


`w` to move word by word

`b` to move backwards word by word

`W` to move word by WORD

`B` to move backwards WORD by WORD

`e` to jump to the end of a word

`ge` to jump to the end of the previous word

 `E` is like `e` but operates on WORDS

 `gE` is like `ge` but operates on WORDS
 

Use `f{character}` (find) to move to the next occurrence of a character in a line.

Use `F{character}` to find the previous occurrence of a character

Use `t{character}` to move the cursor just before the next occurrence of a character (think of `t{character}` of moving your cursor until that character).

Again, you can use `T{character}` to do the same as `t{character}` but backwards

After using `f{character}` you can type `;` to go to the next occurrence of the character or `,` to go to the previous one. You can see the `;` and `,` as commands for repeating the last character search.

`0`: Moves to the first character of a line

`^`: Moves to the first non-blank character of a line

`$`: Moves to the end of a line

`g_`: Moves to the non-blank character at the end of a line

`}` jumps entire paragraphs downwards

`{` similarly but upwards

`CTRL-D` lets you move down half a page by scrolling the page

`CTRL-U` lets you move up half a page also by scrolling

`/{pattern}` to search forward

`?{pattern}` to search backwards

 `n` to go to the next match

 `N` to go to the previous match
 

Use `gd` to **g**o to **d**efinition of whatever is under your cursor.

Use `gf` to **g**o to a **f**ile in an import.

Type `gg` to go to the top of the file.

Use `{line}gg` to go to a specific line.

Use `G` to go to the end of the file.

Type `%` jump to matching `({[]})`.

`v` for character-wise visual mode

`V` for line-wise visual mode

`C-V` for block-wise visual mode

在 Vim 中，你可以使用一些命令来模拟鼠标滚轮滚动页面而不移动光标。以下是一些常用的命令：

1. **向下滚动一行**：
   ```
   Ctrl + e
   ```
   这将滚动页面向下移动一行，而光标保持在原位。

2. **向上滚动一行**：
   ```
   Ctrl + y
   ```
   这将滚动页面向上移动一行，而光标保持在原位。

3. **向下滚动半屏**：
   ```
   Ctrl + d
   ```
   这将滚动页面向下移动半屏，而光标保持在原位。

4. **向上滚动半屏**：
   ```
   Ctrl + u
   ```
   这将滚动页面向上移动半屏，而光标保持在原位。

5. **向下滚动一屏**：
   ```
   Ctrl + f
   ```
   这将滚动页面向下移动一屏，而光标保持在原位。

6. **向上滚动一屏**：
   ```
   Ctrl + b
   ```
   这将滚动页面向上移动一屏，而光标保持在原位。



