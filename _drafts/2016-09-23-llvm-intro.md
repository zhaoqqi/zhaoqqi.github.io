---
layout: post
title: 学习一点LLVM
categories: [LLVM]
description: This is an introduction to doing research with the LLVM compiler infrastructure. It should be enough for a grad student to go from mostly uninterested in compilers to excited to use LLVM to do great work.
keywords: LLVM
---

以下内容翻译自 [LLVM for Grad Students](http://adriansampson.net/blog/llvm.html)，先贴出来再翻译学习。

> LLVM is a compiler. It’s a really nice, hackable, ahead-of-time compiler for “native” languages like C and C++.
> Of course, since LLVM is so awesome, you will also hear that it is much more than this (it can also be a JIT; it powers a great diversity of un-C-like languages; it is the new delivery format for the App Store; etc.; etc.). These are all true, but for our purposes, the above definition is what matters.
> A few huge things make LLVM different from other compilers:
> * LLVM’s intermediate representation (IR) is its great innovation. LLVM works on a representation of programs that you can actually read if you can read assembly. This may not seem like a great revelation, but it is: other compilers’ IRs tend to be in-memory structures so complicated that you can’t really write them down. This makes other compilers harder to understand and messier to implement.
> * LLVM is nicely written: its architecture is way more modular than other compilers. Part of the reason for this niceness comes from its original implementor, who is one of us.
> * Despite being the research tool of choice for squirrelly academic hackers like us, LLVM is also an industrial-strength compiler backed by the largest company on Earth. This means you don’t have to compromise between a great compiler and a hackable compiler, as you do in Javaland when you choose between HotSpot and Jikes.
