+++
title = "Change tailwind classes with ast-grep"
date = 2026-01-29T18:36:34+08:00
draft = false
description = "Using ast-grep to update tailwind classes across a codebase"
slug = "change-tailwind-classes-with-ast-grep"
tags = ["cli-tools", "ast-grep", "refactoring", "react", "tailwind"]
+++

More [ast-grep](https://ast-grep.github.io/) usage!

This time, I needed to standardise the styling of input and dropdown fields across our application to have the same height. Because of how things developed, we ended up using two different heights in different pages (40px & 36px). needed to standardise on one height, which we decided to be 36px. Because we used [Tailwind CSS](https://tailwindcss.com/), that meant finding instances of the `h-10` class and changing it to `h-9`.

```tsx
const From = () => {
  return (
    <Foo className={{
      container: 'flex flex-col h-10',
      content: 'text-typography-900',
    }}>
      <p className="h-10 w-10">hello world</p>
      <p className="text-sm">bye world</p>
    </div>
  )
}

const To = () => {
  return (
    <Foo className={{
      container: 'flex flex-col h-9',
      content: 'text-typography-900',
    }}>
      <p className="h-9 w-10">hello world</p>
      <p className="text-sm">bye world</p>
    </div>
  )
}
```

Once again, I thought that ast-grep would be the perfect tool for this. I only needed to match the substring `h-10` that exists in a valid class prop, which in React is `className`, then change it to `h-9` if applicable. With an AST, I could do exactly that.

Because of how we define the `className` prop for custom components[^1], we would also have to match string values inside objects that are passed to the prop.

[^1]: We do this to get Tailwind CSS LSP support in the strings that are passed to the `className` prop. Custom components may accept several `className` strings because the component might allow the caller to configure different aspects of the component. By passing an object to the `className` prop, we remove the need to add [additional patterns](https://github.com/tailwindlabs/tailwindcss-intellisense?tab=readme-ov-file#tailwindcssclassattributes) every time we have a new class prop that the LSP should match so that we get autocomplete for Tailwind classes.

These form the requirements for our ast-grep rule, which I've created as such:

```yaml
language: tsx
rule:
  kind: string
  pattern: $CLASSES
  inside:
    kind: jsx_attribute
    regex: "className"
    stopBy: end

constraints:
  CLASSES:
    regex: \b(h-10)\b

transform:
  CHANGE_CLASS: replace($CLASSES, replace='h-10', by='h-9')

fix: "$CHANGE_CLASS"
```

For the rule[^2], we start with the node that we want to match which is a `string` node. This string node must be inside, i.e. a child node of, a `jsx_attribute` node. We use the [`stopBy: end` option](https://ast-grep.github.io/reference/rule.html#stopby) to specify that ast-grep should keep searching all the way down inside the `jsx_attribute` node for the nodes that we want to match. This is necessary because we also pass objects with key-string pairs to the `className` prop, and we want to match these strings as well. The `stopBy: end` option allows us to cover both cases: the `jsx_attribute` value being a string or an object with key-string pairs.

[^2]: [Link to playground](https://ast-grep.github.io/playground.html#eyJtb2RlIjoiQ29uZmlnIiwibGFuZyI6InRzeCIsInF1ZXJ5IjoiY29uc29sZS5sb2coJE1BVENIKSIsInJld3JpdGUiOiJsb2dnZXIubG9nKCRNQVRDSCkiLCJzdHJpY3RuZXNzIjoic21hcnQiLCJzZWxlY3RvciI6IiIsImNvbmZpZyI6Imxhbmd1YWdlOiB0c3hcbnJ1bGU6XG4gIGtpbmQ6IHN0cmluZ1xuICBwYXR0ZXJuOiAkQ0xBU1NFU1xuICBpbnNpZGU6XG4gICAga2luZDoganN4X2F0dHJpYnV0ZVxuICAgIHJlZ2V4OiAnY2xhc3NOYW1lJ1xuICAgIHN0b3BCeTogZW5kXG5cbmNvbnN0cmFpbnRzOlxuICBDTEFTU0VTOlxuICAgIHJlZ2V4OiBcXGIoaC0xMClcXGJcblxudHJhbnNmb3JtOlxuICBDSEFOR0VfQ0xBU1M6IHJlcGxhY2UoJENMQVNTRVMsIHJlcGxhY2U9J2gtMTAnLCBieT0naC05JylcblxuZml4OiBcIiRDSEFOR0VfQ0xBU1NcIiIsInNvdXJjZSI6ImNvbnN0IEZyb20gPSAoKSA9PiB7XG5cdHJldHVybiAoXG5cdFx0PEZvbyBjbGFzc05hbWU9e3tcblx0XHRcdGNvbnRhaW5lcjogJ2ZsZXggZmxleC1jb2wgaC0xMCcsXG5cdFx0XHRjb250ZW50OiAndGV4dC10eXBvZ3JhcGh5LTkwMCcsXG5cdFx0fX0+XG5cdFx0XHQ8cCBjbGFzc05hbWU9XCJoLTEwIHctMTBcIj5oZWxsbyB3b3JsZDwvcD5cblx0XHRcdDxwIGNsYXNzTmFtZT1cInRleHQtc21cIj5ieWUgd29ybGQ8L3A+XG5cdFx0PC9kaXY+XG5cdClcbn0ifQ==)

With just the above rules, we would match on every single `className` prop, which is unnecessary. So I added the [`constraints` option](https://ast-grep.github.io/reference/yaml.html#constraints) to limit the nodes we match to only those that contain the string `h-10`. The configuration means we want to constrain the `CLASSES` variable that we defined in the rule as `$CLASSES` to only match nodes that satisfy the regex pattern `\b(h-10)\b`, which means that the string must have a standalone word (string separated by spaces on both ends) that is `h-10`. The parentheses around `h-10` means to capture the string, but as of writing this, I don't recall why I added them. I found out later as well that this line is probably not necessary either.

We still need to change `h-10` to `h-9`, which is what the `transform` option is for. For all the matched `$CLASSES`, we replace instances of `h-10` with `h-9`. Passing `$CHANGE_CLASS` to the `fix` option simply means to run the transform that we defined.

When running this rule, instead of doing a blanket apply to all matches, I selectively apply the changes as not all instances of the matched `h-10` class needs to be changed. This concludes the change that I needed to make.

In hindsight, I could perhaps have used [ripgrep](https://github.com/BurntSushi/ripgrep) and [sd](https://github.com/chmln/sd) to achieve the same outcome in a simpler manner, because the string `h-10` is quite specific to Tailwind CSS and shouldn't really appear elsewhere (I briefly describe this other method in [First use of ast-grep](../first-use-of-ast-grep)). This would perhaps have been more thorough as well, because it would match class strings that aren't children of `jsx_attribute` nodes with the name "className". In this case, I was still able to achieve the desired outcome, but in the future I also have a better understanding of which tools and methods to use to edit specific text across the codebase.
