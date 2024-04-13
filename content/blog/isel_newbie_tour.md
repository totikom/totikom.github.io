+++
title = "A newbie's tour to LLVM Instruction Selection"

description = "In this post I'm going to explain, how instruction selection is implemented in LLVM, what stages it has and where target-dependent hooks can be triggered."
date = 2024-04-10
draft = true
template = "page_with_toc.html"
[taxonomies]
tags = ["programming", "llvm", "compilers"]
[extra]
show_only_description = true
+++
So you wonder, how does instruction selection works LLVM?

First of all, a quick intro to LLVM facilities.
LLVM-based compiler consists of a set of transformation passes, which operates on [LLVM IR](https://llvm.org/docs/LangRef.html) or [MIR](https://llvm.org/docs/MIRLangRef.html).
MIR can express, native instructions, pseudo-instruction and llvm opcode.
# CodeGenPrepare
# Initialization of SelectionDAG
# DAGCombiner
# Type legalization
# DAG legaization
# Vector legalization
# Instruction selection


