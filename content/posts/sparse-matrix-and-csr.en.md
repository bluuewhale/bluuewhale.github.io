+++
title = 'Sparse Matrix & CSR'
date = '2026-04-01T16:23:49+09:00'
draft = false
translationKey = 'sparse-matrix-and-csr'
slug = 'sparse-matrix-and-csr-en'
aliases = ['/posts/sparse-matrix-and-csr-en/']
description = 'How CSR (Compressed Sparse Row) represents a sparse matrix with just three arrays, values, column_indices, and row_pointers, and why it speeds up sparse matrix-vector multiplication and graph neighbor lookups.'
tags = ['Data Structure', 'Linear Algebra', 'Graph', 'CPU']
categories = ['Data Structures', 'Performance']
+++

CSR (Compressed Sparse Row) comes up constantly in graph computation and storage optimization. It first appeared in a 1977 Yale University report, the Yale Sparse Matrix Package, which is why it's sometimes called the Yale format. It was designed to store and process sparse matrices efficiently.

## What a Sparse Matrix Is

A sparse matrix is one where most of the entries are zero. You run into them across nearly every corner of modern computing: scientific computing, graph theory, machine learning. Real-world data, once you cast it as a matrix, almost always ends up looking like this. Take a social network: even with a million users, any given person typically has around 500 friends. Representing that directly as a matrix would require something on the order of 10^12 entries, roughly 7.3 petabytes. Storage cost balloons far beyond the actual information the matrix carries.

The adjacency matrix is a classic example of this. It represents the connections between nodes in a graph, and a simple example makes the pattern obvious:

```bash
0 ── 1
|    |
3 ── 2
```

That graph becomes this adjacency matrix:

```bash
       node0  node1  node2  node3
node0 [  0     1     0     1  ]
node1 [  1     0     1     1  ]
node2 [  0     1     0     1  ]
node3 [  1     1     1     0  ]
```

As the number of nodes grows, the matrix size grows with the square of that count, and most of what fills it is still zero. Adjacency matrices remain popular anyway, since operations like BFS or computing an ego-net need fast neighbor lookups. The problem is storing that matrix as-is wastes enormous amounts of memory, and that's exactly the gap CSR closes.

## The CSR Structure

CSR represents a sparse matrix using just three arrays.

| Name | Size | Description |
|---|---|---|
| `values` | NNZ (Number of Non-Zeros) | Every non-zero value, stored in row order |
| `column_indices` | NNZ | The column each value belongs to |
| `row_pointers` | Row Count + 1 | Where each row starts, as an index into `values` |

The core idea is simple: a zero carries no information anyway, so don't store it. Keep only the actual value and position of every non-zero entry (NNZ).

## Walking Through the Conversion

Let's redraw the earlier adjacency matrix with distinct values for clarity:

```bash
       node0  node1  node2  node3
node0 [  0     A     K     B  ]
node1 [  C     0     D     E  ]
node2 [  0     F     0     G  ]
node3 [  H     I     J     0  ]
```

First, scan the non-zero entries in row order, building `values` and a `column_indices` array recording which column each value came from.

```bash
values: [A, B, C, D, E, F, G, H, I, J]
column_indices: [1, 3, 0, 2, 3, 1, 3, 0, 1, 2]
```

Next, record where each row's span starts and ends within `values`. Row 0 has 2 entries, row 1 has 3, row 2 has 2, and row 3 has 3, so accumulating those boundaries builds `row_pointers`.

```bash
row 0: column_indices[0:2]  # 2 non-zero entries
row 1: column_indices[2:5]  # 3 non-zero entries
row 2: column_indices[5:7]  # 2 non-zero entries
row 3: column_indices[7:10] # 3 non-zero entries

row_pointers: [0, 2, 5, 7, 10]
```

Those three arrays together are the complete CSR representation.

```bash
values: [A, B, C, D, E, F, G, H, I, J]
column_indices: [1, 3, 0, 2, 3, 1, 3, 0, 1, 2]
row_pointers: [0, 2, 5, 7, 10]
```

The original 4×4 matrix needed 16 cells. CSR needs only values and column info for 10 non-zero entries, plus one pointer per row. That gap widens dramatically as matrices grow larger and sparser.

## Where CSR Actually Pays Off

### Sparse Matrix-Vector Multiplication (SpMV)

CSR shines brightest in sparse matrix-vector multiplication, $y = Ax$. Multiplying a dense matrix requires work proportional to its column count times the vector length. Multiplying a CSR-encoded matrix only requires touching the entries that actually hold a value, exactly NNZ of them. Every multiply-and-add against a zero simply never happens.

### Graph Computation

With `row_pointers` in hand, pulling a node's neighbors is nearly instant: slice `column_indices[row_pointers[n] : row_pointers[n+1]]` and you have them. On top of that, since CSR packs its values into contiguous memory, cache locality is excellent, which keeps CPU cache lines working efficiently. That's exactly why graph operations with frequent neighbor lookups, BFS, ego-net computation, run so much faster on top of CSR. It's also why so many graph processing frameworks adopt CSR as their default storage format.

## Where CSR Falls Short

Nothing comes free, though. Because CSR is array-based, it resists modification. Adding a single new edge between two nodes means inserting a value into the middle of `values` and `column_indices`, and an array insertion like that has to shift every following element over by one, an expensive operation. That makes CSR a strong fit for read-heavy workloads where the underlying graph or matrix doesn't change often, and a weaker one where it does.
