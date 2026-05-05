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

_Time taken: 9m 45s_

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

_Time taken: 12m 10s_

### Session usage

I didn't take measurements inbetween the two prompts. But here are the session usage stats for the two runs:

```
  Session
  Total cost:            $4.12
  Total duration (API):  13m 37s
  Total duration (wall): 16m 22s
  Total code changes:    552 lines added, 0 lines removed
  Usage by model:
      claude-haiku-4-5:  1.6k input, 4.6k output, 343.5k cache read, 63.6k cache write ($0.1384)
     claude-sonnet-4-6:  601 input, 12.0k output, 202.0k cache read, 48.3k cache write ($0.4230)
       claude-opus-4-7:  47 input, 37.6k output, 3.6m cache read, 133.4k cache write ($3.56)

  Current session
  ███████████████████████▌                           47% used

  Current week (all models)
  ███▌                                               7% used
```
