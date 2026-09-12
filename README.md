# Linux Essentials — Learning Notes

Running notes from working through Cisco NetAcad's Linux Essentials course, part of my self-directed cyber security prep before starting my degree. This isn't a summary of the course material — it's what I actually did, got wrong, and figured out along the way, section by section.

## Contents

- [Terminal Fundamentals: Coming from Windows](#terminal-fundamentals-coming-from-windows)
- [Variables and Functions in Bash (vs. Python)](#variables-and-functions-in-bash-vs-python)
- [Archiving vs. Compression: tar, gzip, and zip](#archiving-vs-compression-tar-gzip-and-zip)
- [Working with Text: Redirection, Pipes, and Text Filters](#working-with-text-redirection-pipes-and-text-filters)
- [The vi Text Editor](#the-vi-text-editor)
- [Shell Scripting: From Command to Executable Program](#shell-scripting-from-command-to-executable-program)
- [Computer Hardware from the Command Line](#computer-hardware-from-the-command-line)

---

## Terminal Fundamentals: Coming from Windows

Before starting this course, I'd already built real comfort with Windows PowerShell through several months of Python projects — navigating directories, running scripts, diagnosing PATH issues, fixing permission errors. Starting Linux Essentials wasn't learning "what is a terminal" from scratch — it was learning a new dialect of something I already understood conceptually.

The basics translated quickly: navigating directories, running and deleting files, moving around the filesystem. The muscle memory from PowerShell (thinking in terms of "where am I, what's here, what do I do to it") carried over almost immediately — only the exact command syntax needed to be relearned.

## Variables and Functions in Bash (vs. Python)

Having a solid grounding in Python variables and functions made this section land much faster than it would have otherwise — I wasn't learning the *concept* of a variable or a function for the first time, just how Bash's syntax expresses ideas I already understood.

A few real syntax differences worth remembering, since Bash looks deceptively similar to Python in places but behaves differently:

- **No spaces allowed around `=` in variable assignment** (`name=value`, not `name = value`) — unlike Python, where spacing is flexible
- **Referencing a variable's value requires a `$` prefix** (`$name`), whereas *assigning* to it does not — a distinction Python doesn't have, since Python always uses the bare name either way
- **Bash functions don't return values the way Python functions do.** A Bash function's "return" is actually an *exit status code* (a number indicating success/failure), not a value handed back to be used in an expression — a genuinely different concept from Python's `return`, and one of the easiest places to get tripped up if you assume Bash functions behave like Python ones

## Archiving vs. Compression: tar, gzip, and zip

**The core distinction:** `tar` archives — it bundles multiple files/folders into a single file, but does not compress anything on its own. Tools like `gzip` and `bzip2` compress, but only work on a single file at a time; they have no concept of bundling multiple files together.

This means compressing multiple files technically requires two separate steps: archive them into one `.tar` file first, then compress that archive. In practice, `tar` has built-in flags that do both in a single command — `-z` for gzip, `-j` for bzip2, `-J` for xz — but these are shortcuts; the same two underlying operations are still happening behind the scenes.

**Lab: gzip vs. tar's built-in flags**
```
gzip my_archive.tar        # compresses the .tar into my_archive.tar.gz
gunzip my_archive.tar.gz   # reverses it back to my_archive.tar
```

Running `gzip` standalone on a `.tar` file produces the exact same result as running `tar -czf archive.tar.gz /path/to/files` in one step. The `-z` flag doesn't do anything new — it's `tar` calling `gzip` internally on your behalf.

**Compression must be reversed with a matching method.** Whatever tool and flag compressed a file, the same tool and flag has to decompress it again — conceptually similar to needing a matching key to reverse an encryption operation.

**zip works differently by design.** It archives and compresses in one built-in step by default, and it creates a compressed copy while leaving the original file untouched — unlike the gzip workflow, which replaces the original. zip's main real-world use case is cross-platform file sharing, since it's natively supported in Windows File Explorer, where `.tar` is not.

## Working with Text: Redirection, Pipes, and Text Filters

### Basic output redirection
```
sysadmin@localhost:~$ echo "Hello World" > mymessage
sysadmin@localhost:~$ cat mymessage
Hello World
```
`>` sends a command's output into a file, overwriting whatever was there before — running `echo "Greetings" > mymessage` again replaced "Hello World" entirely, it didn't add to it.

`>>` appends instead of overwriting:
```
sysadmin@localhost:~$ echo "How are you?" >> mymessage
sysadmin@localhost:~$ cat mymessage
Greetings
How are you?
```
### Standard output vs. standard error, and why redirecting one alone isn't enough

Linux commands actually have two separate output streams: standard output (stdout, stream 1) for normal results, and standard error (stderr, stream 2) for error messages. A plain `>` only redirects stdout — errors still print to the screen:
```
sysadmin@localhost:~$ find /etc -name hosts
/etc/hosts
find: '/etc/ssl/private': Permission denied
```
To send the error somewhere specific, redirect stream 2 explicitly:
```
sysadmin@localhost:~$ find /etc -name hosts 2> err.txt
/etc/hosts
sysadmin@localhost:~$ cat err.txt
find: `/etc/ssl/private': Permission denied
```
Both streams can be sent to separate files at once:
```
sysadmin@localhost:~$ find /etc -name hosts > std.out 2> std.err
```
### Combining both streams into one file: `2>&1`
```
sysadmin@localhost:~$ find /etc -name hosts > find.out 2>&1
sysadmin@localhost:~$ cat find.out
/etc/hosts
find: '/etc/ssl/private': Permission denied
```
**Why the order matters here — this tripped me up until I traced through it properly.** Bash processes redirections left to right, and `2>&1` doesn't mean "redirect stderr to the file named 1" — the `&1` means "wherever stream 1 (stdout) currently points." It's a snapshot of stdout's *current* destination at that exact point in the command, not a live link that updates later.

So `> find.out 2>&1` works because: first stdout gets pointed at `find.out`, *then* `2>&1` copies that same destination for stderr — both end up in the file.

Reversed (`2>&1 > find.out`), it would break: `2>&1` would run first, while stdout is still pointing at the screen, so stderr would get pointed at the screen too. *Then* `> find.out` would redirect stdout to the file — leaving stdout in the file, but stderr still printing to the screen. Order matters because each redirect captures the current state at the moment it runs, not a reference that follows stdout wherever it goes afterward.

### Redirecting input: `<`

Just as `>` sends output to a file, `<` feeds a file in as input:
```
sysadmin@localhost:~$ tr a-z A-Z < myfile
WOW, I SEE NOW
THIS WORKS!
```
(`tr` translates characters — here, lowercase to uppercase. Run without redirection, `tr` waits for you to type input directly, converting each line as you enter it.)

### Piping commands together

A pipe (`|`) feeds one command's output directly into another command's input, without needing an intermediate file:
```
sysadmin@localhost:~$ cut -d: -f1 /etc/passwd | sort | more
```
Reading this left to right: `cut -d: -f1 /etc/passwd` pulls just the first colon-separated field (the username) out of every line of `/etc/passwd`. That list gets piped into `sort`, which alphabetizes it. The sorted result gets piped into `more`, which pages the output so it doesn't all scroll past at once.

### Paging long output

`more` (and the fuller-featured `less`) display long output one screen at a time, rather than dumping everything at once and scrolling past what you wanted to read:
```
sysadmin@localhost:~$ ls -l /etc | more
...
--More--
```
Useful for anything with more output than fits on one screen — like reading `/etc/passwd` directly, or a long directory listing.

## The vi Text Editor

`vi` has two distinct modes, and mixing them up is the single most common source of confusion when first using it:

- **Command mode** (the default when you open a file) — keystrokes are interpreted as commands (move the cursor, delete, copy, search), not typed text
- **Insert mode** (entered with `i`, `a`, `o`, or `O`) — keystrokes are typed directly into the file as text

`Esc` always returns to command mode from insert mode — the reflex worth building is "if I'm not sure what mode I'm in, hit Esc first."

**Movement (command mode):** `h` `j` `k` `l` move left/down/up/right one unit. `w` jumps to the start of the next word, `e` to the end of the current word, `b` to the start of the previous word. A number before a movement repeats it — `3G` jumps to line 3, `8l` moves right 8 characters.

**Deleting and editing (command mode):** `dw` deletes a word, `dd` deletes the whole current line, `x` deletes one character, `X` deletes one character to the left. A number before any of these repeats it — `2dd` deletes two lines, `14x` deletes 14 characters. `u` undoes the last change; a number before it undoes that many changes (`4u`).

**Copy/paste (command mode):** `yw` "yanks" (copies) a word; `p` pastes after the cursor, `P` pastes before it.

**Combining commands:** `cw` deletes a word *and* drops you straight into insert mode to type its replacement — a genuinely efficient combo once it clicks, since it saves a separate delete-then-insert step.

**Joining and searching:** `J` joins the current line with the next. `/word` searches forward for `word`, `?word` searches backward; `n` repeats the search in the same direction, wrapping around the document if it reaches the end.

**Entering insert mode, four different ways, each starting somewhere different:** `i` inserts before the cursor, `a` inserts after it, `o` opens a new blank line below the current one, `O` opens a new blank line above it.

**Search and replace across the whole file:** `:%s/old /new /g` — this is standard `sed`-style substitution syntax (`%` = whole file, `s` = substitute, `g` = every occurrence on each line, not just the first).

**Saving and quitting:** `:w` saves without quitting. `:wq` (or `:x`, or `ZZ` with no colon) saves and quits. `:q!` quits and discards any unsaved changes — the `!` is what forces it to ignore the fact that changes exist. `:wq!` forces a save even to a read-only file, where permissions allow it.

## Shell Scripting: From Command to Executable Program

A shell script takes a sequence of commands you'd otherwise type one at a time and turns it into a saved, reusable file — write once, run repeatedly, instead of retyping a routine every time. Wrote these using `nano` rather than `vi` for anything short, since `nano`'s simpler for small, quick scripts — `vi` is worth knowing well, but isn't always the faster tool for the job.

**Every script starts with a shebang line:**
```
#!/bin/bash
```
This tells the system which interpreter should run the file — without it, the system wouldn't know to treat the file's contents as Bash commands specifically.

**A script isn't executable by default — this has to be set explicitly:**
```
sysadmin@localhost:~$ chmod a+x sample.sh
sysadmin@localhost:~$ ls -l sample.sh
-rwxrwxr-x 1 sysadmin sysadmin 74 Sep 9 19:49 sample.sh
```
Before `chmod`, the permissions showed `-rw-rw-r--` — no `x` (execute) flag anywhere. `chmod a+x` adds execute permission for everyone (`a` = all).

**Why `./sample.sh` works but plain `sample.sh` doesn't — a real gotcha worth understanding, not just memorizing the fix:**
```
sysadmin@localhost:~$ ./sample.sh
[runs correctly]
sysadmin@localhost:~$ sample.sh
sample.sh: command not found
```
Linux only looks for a bare command name (like `sample.sh`, with no path) in the directories listed in `$PATH` — it does **not** automatically check the current directory as a security measure, unlike how some systems behave by default. `./sample.sh` sidesteps this entirely by giving an explicit path ("run the file right here"), rather than asking Linux to search `$PATH` for it.

**The actual fix — put the script somewhere `$PATH` already checks:**
```
sysadmin@localhost:~$ echo $PATH
/home/sysadmin/bin:/usr/local/sbin:/usr/local/bin:...
sysadmin@localhost:~$ mkdir bin
sysadmin@localhost:~$ mv sample.sh bin
```
`~/bin` was already listed in `$PATH` by default — creating that folder and moving the script into it meant the script could then be run from anywhere, by name alone, with no `./` needed.

**Reading user input with `read`:**
```
echo "Please enter your age"
read age
```
`read` waits for input and stores whatever's typed into the named variable (`$age`).

**Conditionals — two equivalent forms encountered, `test` and `[ ]`:**
```
if test $age -lt 16
then
echo "You are not old enough to drive."
else
echo "You can drive!"
fi

if [ $age -lt 16 ]
then
echo "You are not old enough to drive."
else
echo "You can drive!"
fi
```
`[ ]` is just shorthand for `test` — both do the same comparison, `[ ]` is more commonly seen in practice.

**A conditional can test the success of any command, not just a numeric comparison** — here, `grep` itself is the condition:
```
if grep $name /etc/passwd > /dev/null
then
echo "$name is on this system"
else
echo "$name does not exist"
fi
```
`grep` succeeds (exits with status 0) if it finds a match, and fails otherwise — `if` is really just checking that exit status, whether the condition looks like a number comparison or not. The output itself is thrown away with `> /dev/null` since only *whether* it matched is needed here, not what it found.

**`while` loops — repeat as long as a condition holds:**
```
while [ $num -le 100 ]
do
echo "$num is not greater than 100"
echo "Please enter a number greater than 100"
read num
done
```
Keeps re-prompting for input until the entered number is actually greater than 100 — the condition is rechecked before every loop, same underlying idea as a Python `while` loop.

**`for` loops — repeat once per item in a list:**
```
sysadmin@localhost:~$ for name in /etc/passwd /etc/hosts /etc/group

do
wc $name
done
```
Runs `wc` (word/line/byte count) once for each filename listed.

**Generating a sequence to loop over, and running a command by name inside a loop:**
```
sysadmin@localhost:~$ for num in `seq 1 12`

do
touch test$num
done
```
`` `seq 1 12` `` generates the numbers 1 through 12; the backticks run that command and substitute its output directly into the `for` line. `touch test$num` creates an empty file per number (`test1` through `test12`) — `$num` inserts the current loop value directly into the filename.

## Computer Hardware from the Command Line

This module was mostly reading (CPU architecture, drivers, video/power hardware) — most of that is general knowledge rather than something to document lab-by-lab. The genuinely useful, keep-worthy part is the set of commands for inspecting hardware from the terminal, and a couple of real conceptual distinctions.

**CPU:**
- `arch` — shows the CPU's basic architecture/family (e.g. 32-bit vs 64-bit)
- `lscpu` — much more detailed CPU info
- **64-bit CPUs can run in either 32-bit or 64-bit mode; 32-bit CPUs cannot run 64-bit software at all.** The distinction isn't just "faster/slower" — it's a genuine ceiling on what software can run at all.

**RAM:**
- `free` — shows memory usage; `-g` or `-m` round the output into GB or MB instead of raw bytes
- **A 32-bit system can only address up to 4GB of RAM**, regardless of how much is physically installed — another example of the 32-bit ceiling, not just a CPU-speed issue
- **Swap space** — disk space used as overflow when RAM fills up, letting the system keep running (more slowly) rather than immediately failing

**Buses and peripherals:**
- Buses connect components together — USB (**U**niversal **S**erial **B**us — the "B" is literally "bus," which I hadn't clocked before) and PCI are the two main ones covered
- `lspci` — lists devices connected via the PCI bus
- `lsusb` — lists devices connected via USB
- **USB is hot-plug** (connect/disconnect while the system is running) — **not all devices are**; cold-plug devices require a shutdown first

**Storage — partitioning and device naming:**
- A partition divides a single physical drive into multiple separate, independently-usable sections — far more common as a deliberate practice in Linux than in Windows
- **MBR vs GPT** — two different partitioning schemes. GPT is newer, supports more partitions per disk, and supports partitions larger than 2TB (a hard limit MBR can't exceed). Tools mirror `fdisk`'s naming: `gdisk`, `cgdisk`, `sgdisk` for GPT disks specifically
- **Device file naming in `/dev` has three parts, each meaning something specific:**
  - Prefix by drive type: `hd` for IDE drives, `sd` for USB/SATA/SCSI drives
  - A letter for drive order: `/dev/hda` is the first IDE drive, `/dev/hdb` the second, and so on
  - A number for the partition: `/dev/sda1` and `/dev/sda2` would be the first and second partitions on the first SATA/USB/SCSI drive
---

*This file is a work in progress and will keep growing as I move through the rest of the course*
