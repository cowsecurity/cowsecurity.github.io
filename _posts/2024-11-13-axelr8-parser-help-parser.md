---
title: 'Axelr8 Progress - Working on Help Parser'
date: 2024-11-13
permalink: /posts/2024/11/platform-progress/
tags:
  - dev
  - updates
---

Made decent progress on the security automation platform. Got the core stuff working - Go backend, Elixir for background jobs, and MongoDB. Pretty happy with how it's coming together.

## Current Focus: Binary Help Menu Parser

Working on automating tool setup. Instead of manual configs, trying to make it read help menus and figure out tool options automatically. First tests with simple tools are promising, but hit some interesting problems with complex ones.

### Problems I'm Tackling
- AWS CLI ( or other similar tools ) have this nested help structure that's... fun
- Some help menus are huge (context limits, yay)
- Tools really need to standardize their help formats

### Next Steps
- Test with more complex tools
- Figure out a cleaner way to handle nested commands
- Make the setup process faster

That's all for today. Back to staring at help menus and wondering why they're all different.