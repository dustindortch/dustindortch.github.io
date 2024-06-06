---
layout: post
title: Terraform Provider Functions
---

# Terraform Provider Functions

Have you ever wished for a function to exist in Terraform but it simply wasn’t there? Fret no more! Terraform 1.8 was released last week and includes the general availability of [Provider Functions](https://www.hashicorp.com/blog/terraform-1-8-improves-extensibility-with-provider-defined-functions). This feature allows you to write your own functions in Go and use them in your Terraform configuration. This is a game changer for Terraform and I’m excited to see what the community comes up with.

I decided to try this out for myself using inspiration from the [fibonacci  sequence and memoization that I wrote about with respect to Python](https://dustindortch.com/?page_id=1406). I cloned the [hashicorp/terraform-provider-scaffolding-framework repository](https://github.com/hashicorp/terraform-provider-scaffolding-framework) and got to work.

My initial intent was just to write a function called “fib” that would generate the fibonacci sequence number given an input. It took me down a rabbit hole learning even more about the type system implemented in HCL. The tutorial provided by HashiCorp walks through creating the “hashicups” provider. I loosely used it as a starting point, but I quickly lost interest because they walk you through, line by line, modifying the “template”. No thanks. I wrote a Bash script called [scaffold-terraform-provider.sh](https://gist.github.com/dustindortch/0df94248b232c9b16f757313c2690225) to handle that.

```bash
./scaffold-terraform-provider.sh math
```
