+++
title = 'SIMD JSON: Unlocking Maximum Performance for JSON Deserialization'
date = '2025-09-22T00:00:00+09:00'
draft = false
slug = 'simd-json'
description = 'Overview of the SIMD JSON algorithm that uses SIMD instructions to speed up JSON parsing 2–5x, covering structure indexing and parsing stages'
categories = ['Parsing', 'Performance']
tags = ['SIMD', 'JSON', 'Parsing', 'Performance']
+++

# Limitations of Traditional Scalar State Machine Parsers

## Overview
Traditional JSON parsers are scalar state machines that scan input byte by byte, driving many conditional branches and memory lookups. This leads to branch mispredictions and poor cache locality.

## How a Scalar Parser Works (Pseudo-code)
```
parse(buf):
  i = 0; stack = []
  inStr = false; esc = false
  root = null; key = null

  while i < n:
    c = buf[i]
    if inStr:
      if esc: str.append(c); esc = false
      else if c == '\\': esc = true
      else if c == '"': inStr = false; emit_string()
      else: str.append(c)
      i++; continue

    if is_ws(c): i++; continue
    if c == '"': inStr = true; str = ""; i++; continue
    if c == '{': obj = {}; attach(obj); stack.push(obj); i++; continue
    if c == '[': arr = []; attach(arr); stack.push(arr); i++; continue
    if c == '}': stack.pop(); i++; continue
    if c == ']': stack.pop(); i++; continue
    if c == ':': expect_value = true; i++; continue
    if c == ',': key = null; i++; continue
    if starts_lit(buf,i,"true/false/null"): attach(literal); i+=len; continue
    if is_num_start(c): num,cons = parse_number(buf,i); attach(num); i+=cons; continue
    error()
  return root
```

## Drawbacks of Scalar Parsers
1) **Many branches**  
   Frequent if/switch paths cause branch prediction misses and pipeline stalls.

2) **Branching for string boundaries**  
   Escapes (`\`) and quote handling require extra branching to decide string start/end.

3) **Sporadic memory access**  
   Byte-by-byte checks lead to poor locality and cache misses (L1/L2).

---

# SIMD JSON: What Is It?

SIMD JSON accelerates JSON parsing by using SIMD (Single Instruction, Multiple Data) instructions. Introduced in the 2019 paper *“Parsing Gigabytes of JSON per Second”* (Geoff Langdale, Daniel Lemire), it delivers roughly 2–5× speedups over scalar parsers.

## SIMD Basics
- **SIMD**: Single instruction applied to multiple data lanes in parallel.  
- **Benefits**: Fewer instructions/branches, better cache and bandwidth use.  
- **CPU units**:  
  - x86: SSE/AVX/AVX2/AVX-512 (XMM/YMM/ZMM)  
  - ARM: NEON/SVE (Q/N/Z)

## Why SIMD JSON Is Fast
- **Vectorized operations**: Processes data in wide chunks (e.g., 256 bits) instead of byte-by-byte.
- **Branchless design**: Relies on bitwise vector ops, minimizing branch prediction failures.

---

# SIMD JSON Algorithm

Two stages:
1. **Stage 1: JSON Structure Indexing**
2. **Stage 2: JSON Parsing**

## Stage 1: Fast Structure Indexing (Pseudo-code)
```
for each vector block:
  B  = (block == '\\')
  OD = calc_OD(B)                         // odd-length backslash ends
  Q  = (block == '"')
  Q  = Q & ~OD                            // remove escaped quotes
  R  = clmul(Q, all_ones)                 // in-string mask
  S  = classify([{]}:,) & ~R              // structural char mask
  W  = classify(whitespace)               // whitespace mask
  P  = ((S | W) << 1) & ~W & ~R           // pseudo-structural starts
  S  = (S | P) & ~(Q & ~R)
  extract_indices_from_bitset(S, out_indices)
```

### Key Steps
- Build masks for backslashes (B), escaped-quote ends (OD), valid quotes (Q), and in-string ranges (R via carry-less multiply prefix XOR).
- Mask out structural/whitespace inside strings; include quote positions into structural mask.
- Derive pseudo-structural starts (P) from positions after structural/whitespace, excluding whitespace and in-string bits.
- Combine into final structural mask (S), drop quote boundaries, then extract set-bit indices (token boundaries) using `tzcnt`, often with loop unrolling.

## Stage 2: JSON Parsing (Pseudo-code)
```
for i in out_indices:
  switch chr[i]:
    case '"':          parse_string_to_utf8(); write_tape_str()
    case '-', '0'..'9':parse_number();        write_tape_num()
    case 't','f','n':  parse_atom();          write_tape_atom()
    case '{','[':      push_scope();          write_open_with_link_slot()
    case '}',']':      close_scope_and_backpatch_link()
```
Stage 1 provides token boundaries; Stage 2 walks them and dispatches to the appropriate parsers.

---

# Open Source SIMD JSON Libraries
- Available in C++, Go, Java, etc.

## Caution: SIMD VL Mismatch
- Graviton 4 (SVE/SVE2, 128-bit VL) vs SIMD JSON Java lib (currently 256/512-bit only; 128-bit PR pending).
- Works on Graviton 3 or typical AMD; not on Graviton 4 until 128-bit support lands.

---

# References
- https://arxiv.org/pdf/1902.08318
- https://simdjson.org/api/0.6.0/md_doc_ondemand.html
- https://m.blog.naver.com/ektjf731/223053617175?recommendTrackingCode=2
- https://m.blog.naver.com/ektjf731/223056327958?recommendTrackingCode=2
- https://www.officedaytime.com/simd512e/simdimg/pclmulqdq.html
- https://www.google.com/url?sa=i&url=https%3A%2F%2Fftp.cvut.cz%2Fkernel%2Fpeople%2Fgeoff%2Fcell%2Fps3-linux-docs%2FCellProgrammingTutorial%2FBasicsOfSIMDProgramming.html&psig=AOvVaw3AuytgbRLjE84lJGyH9AoY&ust=1758635610339000&source=images&cd=vfe&opi=89978449&ved=0CBgQjhxqFwoTCOjsvqfC7I8DFQAAAAAdAAAAABAE