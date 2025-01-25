---
title: 'Axelr8 Progress - Some updates'
date: 2025-01-25
permalink: /posts/2025/01/axelr8-updates/
tags:
  - dev
  - updates
---

Yo, It's been some time since I last wrote. Quite a few updates, so let me dump them all here.

About that help menu parser from my last post - I ended up building a whole FastAPI service around it. Pretty neat setup actually. When you hit the API, it fires up an EC2 instance (yeah, temporary one, keeps costs down), grabs whatever docker image you throw at it, and does its thing. Basically runs the --help command and shoots that output straight to Gemini. Gemini's cool cause it spits back exactly the JSON format we need. That JSON then goes into our DB, and the Go service picks it up from there. Working smooth so far.

Got some branding news too. Ditching the name axelr8, going with vallumflow instead. Nothing fancy behind it - axelr8 domains are stupid expensive right now. vallumflow was available and honestly, it's growing on me. Plus, way cheaper, so there's that.

Did some infrastructure stuff too. Hooked up kafka with our go setup, then wrote this new service in elixir. Pretty interesting actually - it's like this universal translator for security tools. You feed it JSON output from whatever security tool you're using, and it figures out how to pull metrics from it. Don't matter what format it comes in, it'll try to make sense of it. Still working on making it better, but the basic idea's working.

Oh, and found this annoying bug while all this was happening. So the rce-agent was acting up with large lines. Like, completely choking on them. Dove into the code (fun times) and turns out the rce-agent uses this gocmd/cmd lib that's got this tiny 64kb buffer limit. First thought was easy - bumped it up to 1MB. But now we've got this other problem where it sits there waiting for the buffer to fill up before it does anything. Kind of a pain. Been poking at it trying to find a fix that doesn't suck. Got some ideas, but nothing solid yet.

That's what I've been up to. More updates when I figure out this buffer thing or if anything else interesting pops up.
