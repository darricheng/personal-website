+++
title = 'Teleterm Chat - Part 0'
date = 2024-01-25T19:37:03+08:00
draft = true
series = 'Teleterm Chat'
tags = ["terminal", "rust"]
+++

I want to build a terminal application for myself to solve a use case that I have. (I'm also using this use-case as an excuse for me to learn and write more Rust!) However, instead of just building it, I want to try something new. Going with my [theme for 2024](/posts/theme-for/2024), I want to try writing about my process of building the app as well. More creating, so I'll code AND write!

This is not a tutorial or guide for how to build a terminal application with Rust. Instead, I'll be documenting my journey of learning how to build a terminal app from scratch as someone with little to no experience doing so. Hopefully it might inspire you to go out there to build stuff as well, or maybe it might scare you away. Who knows! Anyway, you can follow along my journey on [GitHub](https://github.com/darricheng/teleterm-chat) as well.

So what will I be building?

I call it Teleterm Chat[^1]. The basic idea is that I want to easily reply a single telegram chat directly from the terminal, which in this case is mainly my partner. I envision two main functions that I'll need, which I have described in the snippet below. I'll name the command `chat` for now.

[^1]: I just combined telegram and terminal, then just added the main function I want the app to have, which is chat.

```sh
# send a message to the chat
chat "Hello, how was your day?"

# view the chat history, with an optional arg for the number of messages
chat --history 2

# Sample output
You: Hello, how was your day?
Partner: Good. How was yours?
```

You'll notice that the command directly interacts with my chat of choice; I shouldn't have to do any form of authentication to send a message to the chat.

How will you handle the authentication then?

I'm glad you asked! I have no idea! But I think that's part of the fun of building something, that I get to learn in the process of trying to achieve my desired end goal.

My initial thoughts is to have a separate CLI app that handles all the details with telegram, including authentication. We then use this app to generate a shell script with a custom alias that you can set to achieve the outcome that I described above, i.e. the `chat` command, except you can rename the command to be anything you like. I'll probably alias mine to my partner's initials, but you could also use cringier (or sweeter, depending on who you ask) words, such as `honey` or `laopo`[^2].

[^2]: Laopo is the romanisation for the Chinese word wife (老婆).

And that's about it for part 0! I have my final goal. There's plenty that I don't know, so next up will be plenty of figuring out to do. The first would probably be trying to figure out authentication, but we'll see where I end up.

_By the way, if you were wondering why I think this is possible, it's because there are other existing open-source telegram clients out there, such as [Paper Plane](https://github.com/paper-plane-developers/paper-plane) and [Unigram](https://github.com/UnigramDev/Unigram). Telegram also open sourced their library for building client applications called [TDLib](https://core.telegram.org/tdlib/docs/)._
