+++
title = "What happens after rustc? An overview of LLVM compiler backend"

date = 2024-04-08
draft = true
template = "page_with_toc.html"
[taxonomies]
tags = ["programming", "rust", "llvm", "compilers"]
+++
# Preamble
I'm working on LLVM-based compiler backend for about a year.
While reading articles about `rustc`, I've noticed that people tend to focus only frontend part of the compiler (that is, `rustc` is actually only a compiler frontend).
Sometimes, middle-end (or target-independent optimizer) is mentioned, but backend is often _leaved as en exercise for the reader_.

I think, that understanding, what actual compiler backend does and what optimizations it performs will allow the reader to better understand `rustc` and other LLVM-based compilers.

# A short overview of the compilation process

# LLVM Backend
## Instruction selection
## Register allocation
## Instruction Scheduling
## Code generation

# Some interesting backend-specific optiizations

