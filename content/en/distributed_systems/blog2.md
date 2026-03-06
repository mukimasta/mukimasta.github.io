---
author : ['Hanlin']
title: "Distributed System Blog – Google File System"
description: "Design of Writing Process and Context of GFS"
date: 2026-03-03
lastmod: 2026-03-03
type: post
draft: false
translationKey: distributed_systems_blog2
coffee: 1
tags: ['distributed_systems']
categories: ['distributed_systems']
---

# Distributed System Blog – Google File System

**Design of Writing Process and Context of GFS**

## Part 1  Design of Writing Process

Google File System is designed to store numerous data (crawled websites, logs, etc.) on thousands of cheap machines. There are three roles: Master, Chunkserver, and Client. Files are divided into 64MB chunks; every chunk has three copies on different Chunkservers. The writing process is: Master selects a Primary among the Chunkservers of a chunk and assigns a lease (60 seconds) to the Primary. Clients get chunk addresses from the Master and send data to the cache of Chunkservers. The Primary decides the writing sequence and sends a writing signal. Chunkservers then write data permanently in the same sequence.

## Part 2  Context of GFS

In 2003, Google's business focused on the search engine. The workload of the file system was mainly sequential append writing and sequential reading of large files (web crawling, logging, MapReduce input). Therefore, GFS solved problems with simple solutions rather than introducing all distributed features; for example, GFS did not ensure linearizability. I am inspired by the design philosophy: starting from the practical scenario, choose the techniques that are most adaptive but not most general; what we need and do not need; how to trade off between consistency and performance. In 2026, the business of Google has extended a lot. They store data not only for websites but also for various kinds of business (Google Docs, Google Calendar, etc.). In 2010, Google replaced GFS with Colossus, which supports almost all Google services nowadays.
