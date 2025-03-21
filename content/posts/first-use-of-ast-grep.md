+++
title = "First use of ast-grep"
date = 2025-03-21T19:26:34+08:00
draft = false
description = "I finally found a first good use of ast-grep!"
slug = "first-use-of-ast-grep"
tags = ["cli-tools", "ast-grep", "refactoring"]
+++

For a while now, I've been intrigued by [ast-grep](https://github.com/ast-grep/ast-grep), "a fast and polyglot tool for code structural search, lint, rewriting at large scale". Specifically, I'm interested in how ast-grep may be used to do rewrites of code with its awareness of the abstract syntax tree (AST).

I'm happy to share that I was finally able to make my first good use of the tool, and it was as cool as I thought in my head!

## The Problem

In MUI v6, the original Grid component was deprecated in favour of Grid2, so we wanted to upgrade our use of Grid across the codebase.

In the past, I would handle such changes with a combination of [ripgrep](https://github.com/BurntSushi/ripgrep) and [sd](https://github.com/chmln/sd) by running `rg 'Grid' -l | xargs sd -F 'Grid' 'Grid2'`. `rg 'Grid' -l` searches for all instances of the string 'Grid', then lists each file that contains at least one instance on each line in stdout. Each line is piped to `sd` using `xargs`, which with the `-F` flag does a string literal replace of 'Grid' to 'Grid2'. It worked fine for most changes, so I didn't have a good reason to learn ast-grep.

However, this time my old method wouldn't work, because there were many other uses the string 'Grid' across the codebase, such as part of another variable name (e.g. 'DataGridWithPagination' or 'dataGridProps'). I couldn't use regex either, because uses of the `Grid` variable could be in import statements (`import { Grid, Box } from '@mui/material'`), in jsx (`<Grid>{children}</Grid>`), or as standalone variable uses (`const StyledGrid = styled(Grid)`)[^1].

[^1]: Maybe you could, but I'm nowhere near enough of a regex expert to figure out a pattern that would match only the required strings to change.

So I turned to ast-grep for help.

## The Solution

Since ast-grep understands the AST of the given file, we can use it to change only the specific uses of the string 'Grid' that we want, which is only variables called 'Grid'. Here's the contents of the [rules yaml file](https://ast-grep.github.io/guide/rule-config.html) that I used for this change.

```yaml
# grid2_upgrade.yml

id: grid2_upgrade
language: tsx
rule:
  all:
    - kind: identifier
    - regex: ^Grid$
fix: "Grid2"
```

- `id`: Just a unique name for the rule.
- `language`: Since we're editing `.tsx` files, [we use tsx instead of typescript](https://ast-grep.github.io/catalog/tsx/) as our language.
- `rule.all`: Rules are how we should capture the string that we want. `all` means the string should match all the nested rules.
  - `kind`: The kind of AST node that we want. I used the [ast-grep playground](https://ast-grep.github.io/playground.html) to find out the name of the node kind to target. `identifier` means we match only variables.
  - `regex`: The node kind we want must also match this regex pattern, which is only 'Grid'. This is necessary because we also have other identifiers that contain the string 'Grid', such as 'DataGridWithPagination' or 'dataGridProps'.
- `fix`: What to change the matched strings to.

I then run the command `sg scan -r grid2_upgrade.yml path/to/directory/ -U` which updates only the uses of the MUI Grid variable to Grid2.

That's it!

Thanks to ast-grep's understanding of the AST, I was able to target only the specific uses of the 'Grid' string in the codebase that I wanted. No need to individually go through each instance of 'Grid' that occurs in the codebase to check if it's an MUI Grid variable before changing it to 'Grid2'.
