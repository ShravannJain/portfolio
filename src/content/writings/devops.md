---
title: "Exploring DevOps and Conquering SSH Challenges"
date: 2024-12-06
description: "Kicking off the writing feed."
tags: ["devops", "terraform", "ssh"]
---
So, I was working on an SSH Docker container the other day, trying to log in as a root user. But guess what? It just wouldn’t work. As a beginner, I had no clue what the problem was. I wasn’t some pro debugger or anything just fumbling around trying to figure out what the heck was going on.

After messing around for a while, I found the issue the sshd_config file. By default, root access was denied. I manually changed the setting to allow root login, saved it, restarted the setup, and bam! It worked! I was so happy like, “Finally, I did it!” kind of happy.

Felt good tbh, I decided to keep going with my learning. I started exploring Infrastructure as Code (IaC) and came across two tools, Ansible and Terraform. Both seemed super powerful for managing infrastructure, but I focused more on Terraform. I spent the rest of the day learning its basics and getting a feel for how it works.

By the end of it all, I felt like I’d made some solid progress. Fixing that SSH issue and diving into IaC was such a good mix of hands on work and discovery. It was one of those days where you feel like you’ve genuinely learned something. Can’t wait to see what’s next on this journey!