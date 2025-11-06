+++
title = "Using ast-grep to restructure a React prop"
date = 2025-11-06T21:14:34+08:00
draft = false
description = "I explore using ast-grep to perform a more involved change of restructuring a React prop."
slug = "using-ast-grep-to-restructure-a-react-prop"
tags = ["cli-tools", "ast-grep", "refactoring", "react"]
+++

In [First use of ast-grep](../first-use-of-ast-grep), I wrote about how I made use of [ast-grep](https://ast-grep.github.io/) to rename a variable throughout a codebase that I couldn't do with just [ripgrep](https://github.com/BurntSushi/ripgrep) and [sd](https://github.com/chmln/sd). In that case, I reached out to ast-grep because the text I wanted to change was a variable only, and ast-grep gave me the ability to target only variables.

This time, I wanted to restructure a prop passed to a specific component in our React codebase. Below is a simplified example of this.

```tsx
const ParentComponent = () => {
  return (
    <ChildComponent
      // Convert from this prop...
      before={{
        target: "foo",
      }}
      // ...to this prop
      target="foo"
    />
  );
};
```

Again, I thought that ast-grep would be perfect for this. Because it has an awareness of the abstract syntax tree (AST), we can use it to extract the exact string value `"foo"` from within the object value of the `before` prop and convert it to the new `target` prop value.

I spent quite a bit more time figuring out how to make the exact change compared to what I did before, because this change was significantly more complex. I needed to match an entire branch of the syntax tree, yet only extract a sub-section of that branch. The rule file I ended up with is as follows.[^1]

```yaml
id: restructure-prop
language: tsx
rule:
	# Top-level rule should match the entire AST node to change
  all:
		  # `patterns` create variables to use when rewriting the tree
    - pattern: $PROP # this is unused in this case
    - kind: jsx_attribute
    - has:
        all:
          - kind: property_identifier
          - regex: "^before$"
    - has:
        all:
          - kind: jsx_expression
          - has:
              all:
                - kind: object
                - has:
                    all:
                      - kind: pair
                      - has:
                          all:
                          # pattern deep in the tree is possible
                            - pattern: $TARGET_STR
                            - kind: string
    - inside:
        all:
          - kind: jsx_self_closing_element
            has:
              all:
                - kind: identifier
                - regex: "^ChildComponent$"

# fix changes the top-level matched node
fix: target=$TARGET_STR
```

## Understanding the rule

The top-level rule has several requirements to match. If we go down the list in order, the top-level node is a `jsx_attribute` node which has a `property_identifier` node with the value matching only the string "before". This means the top-level node is a prop named "before" that is being passed to a component.

The next requirement is a long list of `has` and `all` rules that ends with the pattern `$TARGET_STR` and kind `string`. This deep nesting is to precisely match the `target` property's value in the `before` prop to the `$TARGET_STR` variable. After using this rule, I learnt that we can simplify the rule by using the [`stopBy` rule](https://ast-grep.github.io/reference/rule.html#stopby), so that we don't have to write every single level of nesting. Also, I learnt that we don't need the `all` rule at every level too. At the end of the article, I will share a much simpler rule that also works for this specific use case.

The last requirement makes sure that the prop we're targeting is a descendent of a component called `ChildComponent`, and only self-closing components are matched. So `<ChildComponent />` is matched, while `<ChildComponent></ChildComponent>` is not.

Because the matched node is the `jsx_attribute` node, i.e. the entire prop and its values, the fix applies to the entire prop, so we can change it to exactly what we need. As we used the `$TARGET_STR` pattern to match out the value of the `target` property that we need, we can use it in the `fix` section to ensure that we get the same value applied to the new `before` prop no matter what that value is.

## Improvements

```yaml
id: restructure-prop
language: tsx
rule:
  kind: jsx_attribute
  all:
    - has:
        kind: property_identifier
        regex: "^before$"
    - has:
        kind: string
        stopBy: end
        pattern: $TARGET_STR
    - inside:
        kind: jsx_self_closing_element
        has:
          kind: identifier
          regex: "^ChildComponent$"

fix: target=$TARGET_STR
```

As mentioned in the previous section, there are ways to simplify the initial rule that I used[^2]. In creating another rule, I came across the [`stopBy` rule](https://ast-grep.github.io/reference/rule.html#stopby) while searching for ways to not need to write every single level of nesting, because I wanted to match something that had a much deeper nesting than the rule in this article. Using `stopBy`, we can greatly simply the nested chain to just a single level, because this rule will stop at the end which is what we need: the string value.

[^1]: [Link to ast-grep playground of initial rule](https://ast-grep.github.io/playground.html#eyJtb2RlIjoiQ29uZmlnIiwibGFuZyI6ImphdmFzY3JpcHQiLCJxdWVyeSI6ImNvbnNvbGUubG9nKCRNQVRDSCkiLCJyZXdyaXRlIjoibG9nZ2VyLmxvZygkTUFUQ0gpIiwic3RyaWN0bmVzcyI6InNtYXJ0Iiwic2VsZWN0b3IiOiIiLCJjb25maWciOiJpZDogcmVzdHJ1Y3R1cmUtcHJvcFxubGFuZ3VhZ2U6IHRzeFxucnVsZTpcblx0IyBUb3AtbGV2ZWwgcnVsZSBzaG91bGQgbWF0Y2ggdGhlIGVudGlyZSBBU1Qgbm9kZSB0byBjaGFuZ2VcbiAgYWxsOlxuXHRcdCAgIyBgcGF0dGVybnNgIGNyZWF0ZSB2YXJpYWJsZXMgdG8gdXNlIHdoZW4gcmV3cml0aW5nIHRoZSB0cmVlXG4gICAgLSBwYXR0ZXJuOiAkUFJPUCAjIHRoaXMgaXMgdW51c2VkIGluIHRoaXMgY2FzZVxuICAgIC0ga2luZDoganN4X2F0dHJpYnV0ZVxuICAgIC0gaGFzOlxuICAgICAgICBhbGw6XG4gICAgICAgICAgLSBraW5kOiBwcm9wZXJ0eV9pZGVudGlmaWVyXG4gICAgICAgICAgLSByZWdleDogXCJeYmVmb3JlJFwiXG4gICAgLSBoYXM6XG4gICAgICAgIGFsbDpcbiAgICAgICAgICAtIGtpbmQ6IGpzeF9leHByZXNzaW9uXG4gICAgICAgICAgLSBoYXM6XG4gICAgICAgICAgICAgIGFsbDpcbiAgICAgICAgICAgICAgICAtIGtpbmQ6IG9iamVjdFxuICAgICAgICAgICAgICAgIC0gaGFzOlxuICAgICAgICAgICAgICAgICAgICBhbGw6XG4gICAgICAgICAgICAgICAgICAgICAgLSBraW5kOiBwYWlyXG4gICAgICAgICAgICAgICAgICAgICAgLSBoYXM6XG4gICAgICAgICAgICAgICAgICAgICAgICAgIGFsbDpcbiAgICAgICAgICAgICAgICAgICAgICAgICAgIyBwYXR0ZXJuIGRlZXAgaW4gdGhlIHRyZWUgaXMgcG9zc2libGVcbiAgICAgICAgICAgICAgICAgICAgICAgICAgICAtIHBhdHRlcm46ICRUQVJHRVRfU1RSXG4gICAgICAgICAgICAgICAgICAgICAgICAgICAgLSBraW5kOiBzdHJpbmdcbiAgICAtIGluc2lkZTpcbiAgICAgICAgYWxsOiBcbiAgICAgICAgICAtIGtpbmQ6IGpzeF9zZWxmX2Nsb3NpbmdfZWxlbWVudFxuICAgICAgICAgICAgaGFzOlxuICAgICAgICAgICAgICBhbGw6XG4gICAgICAgICAgICAgICAgLSBraW5kOiBpZGVudGlmaWVyXG4gICAgICAgICAgICAgICAgLSByZWdleDogXCJeQ2hpbGRDb21wb25lbnQkXCJcblxuIyBmaXggY2hhbmdlcyB0aGUgdG9wLWxldmVsIG1hdGNoZWQgbm9kZVxuZml4OiB0YXJnZXQ9JFRBUkdFVF9TVFIiLCJzb3VyY2UiOiJjb25zdCBQYXJlbnRDb21wb25lbnQgPSAoKSA9PiB7XG4gIHJldHVybiAoXG5cdFx0PENoaWxkQ29tcG9uZW50XG5cdFx0XHQvLyBDb252ZXJ0cyBmcm9tIHRoaXMgcHJvcC4uLlxuXHRcdFx0YmVmb3JlPXt7XG5cdFx0XHRcdHRhcmdldDogXCJmb29cIlxuXHRcdFx0fX1cblx0XHRcdC8vIC4uLnRvIHRoaXMgcHJvcFxuXHQgICAgdGFyZ2V0PVwiZm9vXCJcblx0ICAvPlxuICApO1xufTsifQ==).
[^2]: [Link to ast-grep playground of simplified rule](https://ast-grep.github.io/playground.html#eyJtb2RlIjoiQ29uZmlnIiwibGFuZyI6InRzeCIsInF1ZXJ5IjoiY29uc29sZS5sb2coJE1BVENIKSIsInJld3JpdGUiOiJsb2dnZXIubG9nKCRNQVRDSCkiLCJzdHJpY3RuZXNzIjoic21hcnQiLCJzZWxlY3RvciI6IiIsImNvbmZpZyI6ImlkOiByZXN0cnVjdHVyZS1wcm9wXG5sYW5ndWFnZTogdHN4XG5ydWxlOlxuICBraW5kOiBqc3hfYXR0cmlidXRlXG4gIGFsbDpcbiAgICAtIGhhczpcbiAgICAgICAga2luZDogcHJvcGVydHlfaWRlbnRpZmllclxuICAgICAgICByZWdleDogXCJeYmVmb3JlJFwiXG4gICAgLSBoYXM6XG4gICAgICAgIGtpbmQ6IHN0cmluZ1xuICAgICAgICBzdG9wQnk6IGVuZFxuICAgICAgICBwYXR0ZXJuOiAkVEFSR0VUX1NUUlxuICAgIC0gaW5zaWRlOlxuICAgICAgICBraW5kOiBqc3hfc2VsZl9jbG9zaW5nX2VsZW1lbnRcbiAgICAgICAgaGFzOlxuICAgICAgICAgIGtpbmQ6IGlkZW50aWZpZXJcbiAgICAgICAgICByZWdleDogXCJeQ2hpbGRDb21wb25lbnQkXCJcblxuZml4OiB0YXJnZXQ9JFRBUkdFVF9TVFIiLCJzb3VyY2UiOiJjb25zdCBQYXJlbnRDb21wb25lbnQgPSAoKSA9PiB7XG4gIHJldHVybiAoXG5cdFx0PENoaWxkQ29tcG9uZW50XG5cdFx0XHQvLyBDb252ZXJ0cyBmcm9tIHRoaXMgcHJvcC4uLlxuXHRcdFx0YmVmb3JlPXt7XG5cdFx0XHRcdHRhcmdldDogXCJmb29cIlxuXHRcdFx0fX1cblx0XHRcdC8vIC4uLnRvIHRoaXMgcHJvcFxuXHQgICAgdGFyZ2V0PVwiZm9vXCJcblx0ICAvPlxuICApO1xufTsifQ==).
