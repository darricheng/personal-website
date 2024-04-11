+++
title = "Ask For Forgiveness, Not Permission"
date = 2023-10-13T09:40:13+08:00
draft = false
description = "The first time I tried using the quote ask for forgiveness, not permission"
slug = "ask-for-forgiveness-not-permission"
tags = ["javascript", "rust", "performance-optimisation"]
+++

I came across this saying from ThePrimeagen, that as a software engineer, we should not ask for permission to do things. Instead, when we see a potential solution to a problem, create prototypes to show others the solution. Then, if things don't work out, ask for forgiveness. This is in contrast to asking for permission to work on a potential solution before actually getting down to creating anything.

The reason given is that it is difficult for others to fully understand your ideas from an explanation with only text. It is much easier to build a prototype that demonstrates your ideas to others, so that they can see exactly what goes on for your idea. Also, building a prototype validates the idea, so that we don't leave others hanging with a great idea that doesn't actually work.

I recently experienced this, and I think this saying held true for me. In my team, we use JavaScript for everything. Not what I would ideally like to use, but it works for our use cases. There is one part of our service that requires speed; our staff have to generate hundreds of PDFs daily, so any time we can save generating the PDF is multiplied.

Our previous implementation uses HTML templates passed to headless chromium to generate the PDFs. Running a heavyweight process hundreds of times a day isn't the most efficient way to generate PDFs, so I explored other methods, such as using [WeasyPrint](https://weasyprint.org/). However, I couldn't find any that would suit our needs. After a frustrated afternoon or two, I realised what we want is a PDF, regardless of how we generate it. So I explored libraries that could programmatically generate PDFs without relying on HTML templates. I decided to explore a language that I'm currently excited about: Rust. I reasoned that using a low-level language would be faster as well (it was the only low-level language that I felt confident enough with anyway). I was also aware of libraries that allowed for inter-op between Rust and NodeJS, such as [napi-rs](https://napi.rs/), so I settled on that.

By just hard-coding all the values for the generated PDF, I was able to reduce the time needed to generate the PDF by a factor of more than 20! I showed the idea to my team lead, and he gave me the go-ahead to make the implementation ready for production. I would actually get to work with a technology that I was excited about at work! I was stoked. I managed to get the idea ready and ship it to production.

It felt great to have an idea validated first with a prototype, then with subsequent approval and the green light to go ahead with implementation from senior engineers. I gained confidence that I'm able to independently create solutions that add value to my organisation. Perhaps sometime in the future I will be able to prototype entire systems that improve metrics across the board too! I think the best thing about this experience was that I got to work on something that I felt much more excited about compared to the regular work of working on features or fixing bugs, and gives me more hope that I might be able to work on more of such stuff in the future.

In hindsight, I think my excitement to use Rust at my workplace was too great. I should have done more testing to find out whether it was actually faster than just writing the program in JavaScript. The library would be much easier to maintain as the team is familiar with JavaScript, and that might be worth the whatever speed gains using Rust would give (if any at all). Hindsight is 20/20 though, so I learn.
