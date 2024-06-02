+++
title = "i18n CSV <-> JSON"
date = 2024-06-01T18:41:17+08:00
draft = false
description = "Shell scripts to convert i18n data between CSV and JSON formats"
slug = "i18n-csv-to-and-from-json"
tags = ["shell", "csv", "json", "scripting"]
image = "images/i18n-csv-json-meta.jpeg"
+++

I recently worked with the internationalisation library [i18next](https://www.i18next.com/) to add multiple language support to an app. In the process, I found myself needing to convert data between two different formats:

- JSON key-value pairs used by i18next to match keys to their respective translations.
- Spreadsheet data with matching translations to their english counterparts (essentially CSV).

Example JSON data:

```json
{
  "foo": "Translated Foo",
  "bar": {
    "baz": "Translated Baz"
  }
}
```

Corresponding example CSV data:

```csv
foo,Translated Foo
bar.baz, Translated Baz
```

I definitely wasn't going to do the format conversion manually. Besides, this likely wouldn't be the last time I'll be converting the data between the two formats! So I came up with two simple shell scripts to help me.

_Side note: I include links to the tools I use at the bottom of the article. Or you could just search them up with your favourite search engine._

## JSON to CSV (sort of)

The first is to convert the JSON format to CSV so that I could input all the text that needed to be translated into the spreadsheet.

Instead of comma separated values, I create semicolon separated values. Sometimes, the text to be translated would include commas, and splitting the copied text by commas in the spreadsheet software would split the text into more than two columns which isn't what I want (the two columns being the JSON path or key, and the translated text). Semicolons pretty much don't occur in the text, so I used them as my delimiter instead.

```shell
fd -e json translation . -x \
  jq --stream -r 'select(has(1)) | "\(first | join("."));\(last)"' | \
  pbcopy
```

Let's breakdown the one-liner.

> `fd -e json translation . -x`

`fd` is a tool similar to `find` that is available in UNIX. The purpose of this command is to find where our translations JSON file is located, then extract the data to manipulate.

- `-e json`: Tells `fd` that we only want files with the `json` file extension.
- `translation .`: search for files that contain the text "translation" in the current directory.
- `-x`: Execute the command that follows for each search result, which in this case is the `jq` command that we'll cover next.

> `jq --stream -r 'select(has(1)) | "\(first | join("."));\(last)"' | pbcopy`

`jq` is an amazing tool for working with JSON data. I've only barely started to scratch the surface with it. We use `jq` to extract the JSON data and convert it into a format that we want.

- `--stream`: Converts the JSON input data into an array of path and leaf values that we can work with. Our example JSON will be converted into:

```json
[ [ "foo" ], "Translated Foo" ]
[ [ "bar", "baz" ], "Translated Baz" ]
[ [ "bar", "baz" ] ]
[ [ "bar" ] ]
```

- `-r`: Format the output as raw output, i.e. strings won't have quotes like in JSON.
- `'select(has(1)) | "\(first | join("."));\(last)"'`: The `jq` operations that we want to perform on the given input. We'll breakdown the command within the single quotes into two parts. `|` has a similar meaning to when used in a shell, i.e. pass the value from the previous command's output to the next command's input.
  - `select(has(1))`: If the input array (each line from the `--stream` flag above) has a value at index 1, i.e. has text to be translated, select it and pass it to the next command.
  - `"\(first | join("."));\(last)"`: The outer double quotes mean to create a string, with the backslashes used to escape characters. The semicolon is a literal part of the string and has no semantic meaning here.
    - `\()`: Treat the text within the parentheses as commands, then replace the location of the parentheses with the result of the commands.
    - `first | join(".")`: Take the first element of the array and join its array elements into a string with "." as the separator. Turns `["foo", "bar"]` into `foo.bar`.
    - `last`: Return the last element of the array which is the leaf node or text to be translated.
  - With these `jq` flags and input, `{ "foo": { "bar": "baz" } }` will be turned in `foo.bar;baz`
- `| pbcopy`: Pipe the final output to OSX's clipboard, ready to be pasted into the translations spreadsheet.

All that's left is to paste the clipboard value into the spreadsheet, then tell the spreadsheet to separate the pasted values by semicolon into columns!

## CSV to JSON

The second is to convert CSV with the translated text back to JSON so that we can add a new language to i18next.

```sh
# fish shell
touch translations.json
echo '{}' > translations.json
for i in (bat translated-text.csv)
  bat translations.json |
  jq \
    --argjson a (echo $i | xsv select 1 | \
      sed -e 's/\./", "/g' -e 's/^/["/' -e 's/$/"]/') \
    --arg b (echo $i | xsv select 2) \
    'getpath($a)=$b' |
  sponge translations.json
end
```

This time it's a little more involved. Let's start with the first two lines.

- `touch translations.json`: Create a new file.
- `echo '{}' > translations.json`: Pass `{}` as data to stdout using `echo`, then `>` redirects the output to the translations.json file, meaning it will overwrite the file contents with `{}`.

Next is the for loop. In my case, I use fish shell, but any shell's for loop should also work fine.

- `for i in (bat translated-text.csv) ... end`: Iterate through the CSV file, passing each line as data to the variable `i`.
- `bat translations.json`: Extract the current state of the translations JSON file and pipe it to `jq`.
- `jq --argjson a (some command)`: Pass the output of `some command` as a JSON-encoded value to `jq` as the variable `a`.
- `echo $i | xsv select n`: Pass `$i` via stdin to `xsv`, then select column n of the CSV value in `$i`.
- `sed -e 's/\./", "/g' -e 's/^/["/' -e 's/$/"]/'`: Convert the file path to an array, i.e. `some.nested.key` -> `["some", "nested", "key"]`. `-e` allows us to provide `sed` with multiple substitution rules in a single `sed` invocation.
  - `s/\./", "/g`: Replace all occurrences of `.` with `", "`.
  - `s/^/["/`: Add `["` to the start of the line.
  - `s/$/"]/`: Add `"]` to the end of the line.
- `jq --arg b (some command)`: Pass the output of `some command` as a string value to `jq` as the variable `b`.
- `jq 'getpath($a)=$b'`: Set the path indicated by the `jq` variable `$a` to the value of the `jq` variable `$b`. The path needs to be an array of strings, thus the use of `--argjson` instead of just `--arg` for variable a.
- `sponge translations.json`: Updates the `translations.json` file with the path-value pair from the current loop iteration. E.g. `{"foo": "bar"}` to `{"foo": "bar", "nested": {"key": "nestedValue"}}`. `sponge` allows for writing to the same file that we read from earlier (this line started with `bat translations.json`). This method of iteratively updating a single file wouldn't work without `sponge`.

## Some thoughts

I share my experience here with only a secondary purpose of hoping it'll help someone else out there with a similar scripting requirement.

As a software engineer, I believe it is critical for me to understand what goes on when I use a certain tool or write a specific piece of code. Sharing my experience allows me to clarify and solidify my learning of the tools I worked with.

If you read till here, thank you! Arriving at the scripts that I have was a mini adventure for me. It was tough (I struggled with `jq` in particular), but the joy I felt when the scripts worked made it all worthwhile.

## Tools used

- [fish shell](https://fishshell.com/)
- [fd](https://github.com/sharkdp/fd)
- [jq](https://jqlang.github.io/jq/)
- [bat](https://github.com/sharkdp/bat) (just the UNIX builtin `cat` will do)
- [xsv](https://github.com/BurntSushi/xsv)
- sed (UNIX builtin)
- [sponge](https://linux.die.net/man/1/sponge)
