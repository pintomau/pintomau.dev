+++
title = "go-spmc-ring"
date = 2026-06-30
description = "A lock-free single-writer, multiple-reader ring buffer in Go, inspired by the LMAX Disruptor."
[taxonomies]
tags = ["go", "concurrency", "performance"]
[extra]
tech = ["Go", "Lock-free", "SIMD", "Concurrency"]
repo = "https://github.com/pintomau/go-spmc-ring"
status = "active"
+++

A single-writer, multiple-reader (SPMC) ring buffer in Go, inspired by the LMAX Disruptor pattern. It focuses on 
cache-line awareness, false-sharing avoidance, and lock-free reader lifecycle management: **unlike the classic
Disruptor (where the consumer graph is wired once before the first event) readers here join and leave a
live stream at runtime without locks, allocations, or writer stalls.**

This is useful for runtimes and actor model-like systems where you need some entering and reentering readers.

We also preview the compatibility and usage with the preview SIMD package in Go 1.26, which allows for vectorized
operations on the ring buffer.

On a Ryzen 5 9600X (Linux, Go 1.26) a single publish takes ~5.2 ns (≈193 M ops/s), batched publishes reach ~2.1 ns/item
(≈476 M items/s), and eight concurrent readers cost the writer less than 7%.

Check the [repo](https://github.com/pintomau/go-spmc-ring).