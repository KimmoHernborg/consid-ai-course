Task A: Prompting for Repo Analysis

Repositories that we will analyze in this task:

- https://github.com/warpdotdev/warp (a terminal emulator built in Rust)
- https://github.com/zed-industries/zed (a code editor built in Rust and WebAssembly)
- https://github.com/anomalyco/opencode (an open source AI harness)

## Basic prompt for analyzing a repo

The prompt for analyzing the repo in a checked out local copy:

```
Analyze this repo and provide a summary of its purpose, main features, and any notable technologies used. Write the result to a file called `ANALYSIS-1.md`.
```

## Detailed prompt for analyzing a repo

The prompt for analyzing the repo in a checked out local copy:

```
Analyze this repo and provide a detailed report.

1. Architecture: what layers are there, how do they communicate?
2. External dependencies: what libraries and what does each one do?
3. Integrations: databases, APIs, other services?
4. Business logic: what files and functions?
5. Technical debt: what is clearly problematic?
6. How is different platform support implemented?

Also include a conclusion with the following points:
- What the project does and for whom
- The architecture in broad strokes
- The three most important technical dependencies and why they are used
- One thing that surprised you about the structure

Write the result to a file called `ANALYSIS-2.md`.
```
