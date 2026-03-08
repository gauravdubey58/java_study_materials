# Build Tools Notes — Apache Maven & Apache Ant

> A structured, book-style reference guide covering two of the most widely used Java build tools: **Apache Maven** and **Apache Ant**. Designed for developers who want clear, organised notes to study, reference, or upload to GitHub.

---

## What's Inside

This repository is split into two independent books — one per tool. Each book is divided into chapters that progressively build on each other, from core concepts through to advanced usage and real-world patterns.

| Book | Chapters | Topics |
|---|---|---|
| 📘 [Apache Maven](./maven/) | 12 | POM, lifecycles, phases, dependency management, repositories, plugins, profiles, multi-module builds |
| 📙 [Apache Ant](./ant/) | 13 | Build files, targets, tasks, data types, Antlibs, Ivy, macros, control flow |

---

## How to Navigate

- Start at the **[Index](./index.md)** for a full chapter-by-chapter table of contents
- Or jump directly into either book using the links above
- Every chapter page has **← Previous** and **Next →** links at the top and bottom

---

## Who This Is For

- Java developers studying build tools for the first time
- Engineers moving from Ant to Maven (or vice versa)
- Teams that need a shareable internal reference
- Anyone preparing for technical interviews or certifications

---

## Repository Structure

```
build-tools-notes/
├── README.md                   ← You are here
├── index.md                    ← Full table of contents
│
├── maven/                      ← Apache Maven book
│   ├── 01-introduction.md
│   ├── 02-core-concepts.md
│   ├── 03-pom.md
│   ├── 04-lifecycles.md
│   ├── 05-phases-and-goals.md
│   ├── 06-dependency-management.md
│   ├── 07-repositories.md
│   ├── 08-plugins.md
│   ├── 09-profiles.md
│   ├── 10-multi-module.md
│   ├── 11-use-cases.md
│   └── 12-quick-reference.md
│
└── ant/                        ← Apache Ant book
    ├── 01-introduction.md
    ├── 02-core-concepts.md
    ├── 03-build-file-structure.md
    ├── 04-lifecycle-execution.md
    ├── 05-data-types.md
    ├── 06-built-in-tasks.md
    ├── 07-antlibs.md
    ├── 08-properties-and-filtering.md
    ├── 09-conditions-and-control-flow.md
    ├── 10-macros-and-custom-tasks.md
    ├── 11-ivy-dependency-management.md
    ├── 12-use-cases.md
    └── 13-quick-reference.md
```

---

*Apache Maven docs: https://maven.apache.org/guides/*
*Apache Ant docs: https://ant.apache.org/manual/*
