# Linux Essentials — Learning Notes

Running notes from working through Cisco NetAcad's Linux Essentials course, part of my self-directed cyber security prep before starting my degree. This isn't a summary of the course material — it's what I actually did, got wrong, and figured out along the way, section by section.

## Contents

- [Terminal Fundamentals: Coming from Windows](#terminal-fundamentals-coming-from-windows)
- [Variables and Functions in Bash (vs. Python)](#variables-and-functions-in-bash-vs-python)
- [Archiving vs. Compression: tar, gzip, and zip](#archiving-vs-compression-tar-gzip-and-zip)

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

---

*This file is a work in progress and will keep growing as I move through the rest of the course — remaining sections will cover pipes and redirection, basic scripting, package and process management, network configuration, and user/permissions management.*
