# 🚀 Vim Speed Manual for CKAD

A focused, exam-oriented Vim reference designed specifically for the CKAD environment.

This is not a full Vim guide.  
It contains only what increases speed, precision, and confidence under exam pressure.

---

# 1. Opening Files

```bash
vim file.yaml
```

Open an existing file or create a new one.

---

# 2. Vim Modes (Critical Understanding)

| Mode    | Purpose                                      |
|---------|----------------------------------------------|
| Normal  | Navigate, delete, copy, modify               |
| Insert  | Type and edit text                           |
| Visual  | Select multiple lines or blocks              |
| Command | Save, quit, replace, line operations (`:`)   |

Press `Esc` anytime to return to **Normal mode**.

---

# 3. Insert & Append Operations

| Action                         | Command |
|--------------------------------|----------|
| Insert at cursor               | `i` |
| Insert at start of line        | `I` |
| Insert below current line      | `o` |
| Insert above current line      | `O` |
| Append after cursor            | `a` |
| Append at end of line          | `A` |

Exit Insert mode:

```
Esc
```

---

# 4. Saving & Exiting

| Action                 | Command |
|------------------------|----------|
| Save                   | `:w` |
| Quit                   | `:q` |
| Save & Quit            | `:wq` |
| Quit without saving    | `:q!` |
| Save as new file       | `:w newfile.yaml` |

---

# 5. Navigation (Core Speed Skills)

## Line Navigation

| Action                  | Command |
|-------------------------|----------|
| Beginning of line       | `0` |
| End of line             | `$` |
| Go to specific line     | `:25` |

## File Navigation

| Action          | Command |
|-----------------|----------|
| Top of file     | `gg` |
| Bottom of file  | `G` |

## Word Navigation

| Action         | Command |
|----------------|----------|
| Next word      | `w` |
| Previous word  | `b` |
| End of word    | `e` |

---

# 6. Deleting Text Efficiently

Delete current line:

```
dd
```

Delete multiple lines:

```
5dd
```

Delete from current line to end of file:

```
dG
```

Delete entire file:

```
:%d
```

Delete specific line range:

```
:10,20d
```

---

# 7. Visual Mode (Bulk Editing)

Enter line-wise Visual mode:

```
Shift + V
```

Extend selection:

```
j   (down)
k   (up)
```

Delete selection:

```
d
```

Indent selection:

```
>
```

Unindent selection:

```
<
```

---

# 8. Copy & Paste

Copy current line:

```
yy
```

Copy multiple lines:

```
3yy
```

Paste below:

```
p
```

Paste above:

```
P
```

---

# 9. Indentation (Critical for YAML)

Indent current line:

```
>>
```

Indent multiple lines:

```
5>>
```

In Visual mode:

```
>   (indent)
<   (unindent)
```

YAML requires strict indentation. Always use spaces, never tabs.

---

# 10. Search & Replace

Search forward:

```
/image
```

Next match:

```
n
```

Previous match:

```
N
```

Replace in current line:

```
:s/old/new/g
```

Replace in entire file:

```
:%s/old/new/g
```

---

# 11. Editing Shortcuts

Delete word:

```
dw
```

Change word:

```
cw
```

Delete inside quotes:

```
di"
```

Delete inside parentheses:

```
di(
```

Undo:

```
u
```

Redo:

```
Ctrl + r
```

---

# 12. CKAD Workflow Optimization

Avoid writing YAML from scratch unless required.

Generate a resource template:

```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
```

Edit the generated file using Vim.  
This reduces syntax errors and saves time.

---

# 13. Recommended Vim Settings for YAML

Inside Vim:

```
:set number
:set relativenumber
:set tabstop=2
:set shiftwidth=2
:set expandtab
```

Optional: Add to `~/.vimrc`

```
set number
set relativenumber
set tabstop=2
set shiftwidth=2
set expandtab
```

These settings enforce:
- Line numbers
- Relative line numbers
- 2-space indentation
- Spaces instead of tabs

---

# 14. Essential Commands to Memorize

```
i
o
Esc
dd
5dd
yy
p
Shift+V
>
<
gg
G
w
b
$
0
/search
n
dG
u
Ctrl+r
```

---

# Final Principle

CKAD speed comes from:

- Navigating without arrow keys  
- Deleting blocks instantly  
- Using Visual mode efficiently  
- Generating YAML instead of typing manually  
- Maintaining correct indentation every time  

Precision + repetition = speed.
