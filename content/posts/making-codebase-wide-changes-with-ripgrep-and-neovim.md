+++
title = "Making codebase-wide changes with ripgrep and Neovim"
date = 2026-03-17T22:55:17+08:00
draft = false
description = "Using Neovim quickfix to track required changes found with ripgrep in a codebase."
slug = "making-codebase-wide-changes-with-ripgrep-and-neovim"
tags = ["cli-tools", "neovim", "ripgrep"]
+++

Tracking required fixes across a codebase can be tedious and time-consuming. In a large and complex codebase, there are many different directories and file paths to potentially keep note of, and a single file might require multiple changes based on the requirements as well.

I've always thought that Neovim's quickfix list would be perfectly suited to this task. It was just a matter of finding out how to get the locations that required changes into the quickfix list for tracking.

Specifically, there have been several instances where I would be able to find locations that needed edits with [ripgrep](https://github.com/BurntSushi/ripgrep), then needed to make changes at those locations.

Previously, I would keep another terminal open in a tmux pane or window with the list of files that needed changes. But, as it turns out, it's not that difficult to get ripgrep working with Neovim at all! The important flag to know is `--vimgrep`. This flag instructs ripgrep to print results with every match on its own line, including line numbers and column numbers.

There are two ways I found that we can make use of this. I don't think there's much difference using either, I think it's just mainly down to personal preference.

## Piping to Neovim

The first way is to use a Unix pipe.

```bash
rg 'pattern' --vimgrep | nvim -q -
```

This command takes the output from ripgrep and pipes it to Neovim. The important character here is the `-` (dash) after the `-q` flag for the `nvim` command. The dash tells Neovim to read from stdin, which is piped from the stdout of ripgrep via the vertical bar `|` character. Without the dash, Neovim will fail to open with an error.

## Using a tmp file

The second way is to make use of a file to store the output from ripgrep first, then opening the file in Neovim's quickfix list.

```bash
# Create tmp file with errors
rg 'pattern' --vimgrep > /tmp/errorfile
# Use file in nvim quickfix
nvim -q /tmp/errorfile

# fish one-liner
nvim -q (rg 'pattern' --vimgrep | psub)
```

The two-step commands works for both bash and fish (the shell I use). The first step redirects the output to a file in the `/tmp` directory, which means it'll be deleted after some time. We then open the file with Neovim's `-q` flag, which opens the file in the quickfix list directly.

There is only fish one-liner because it's the shell I use, and I wasn't able to get a working one-liner with bash or zsh. The one-liner uses [process substitution](https://fishshell.com/docs/current/cmds/psub.html), where the command in the parentheses before the `|` is essentially passed into a temporary file that is then passed to the calling `nvim -q` command, i.e. pretty much what the two-step does.

## File editing workflow

When we launch Neovim with the above, the quickfix list is populated with a list of entries that each have a file path, line number, and column number. That means each entry is exactly the location of the match found by ripgrep.

When making the edits, I use `]q` to jump to the next location once the fix has been made. I also have `<Leader>q` set up to toggle the quickfix list, so once I'm done with a set of edits, I can quickly jump into the quickfix list to delete the entries that have been edited.

I like to append the `+cwindow` flag to the `nvim` command, so that Neovim opens with the quickfix list already open. It's a minor detail, but it reminds me when Neovim opens that I'm mainly dealing with a quickfix list before my usual habits kick in when opening Neovim (usually opening the file picker immediately). I say minor because opening the quickfix list is a matter of pressing `<Leader>q`.

Interestingly, the quickfix list won't update after you've made edits to the file. It makes sense though, because quickfix is only making use of the line and column numbers to find the right position to place the cursor!
