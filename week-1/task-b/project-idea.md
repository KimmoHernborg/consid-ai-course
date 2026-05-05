# Ideas for a project to implement during the course

## Bookmarking service using AI to tag the links

- Name ideas: **bookm**, **linky**, **hoardy**
- Add single page URLs or bulk import open tabs from browsers
- Use AI to analyze the content of the linked pages and tag them
- Use OpenRouter API to process the links and generate tags for them
- Search and filter links based on the tags and link title
- Implement basic authentication and user accounts using PocketID
- Also create a Chrome extension to easily add links to the site
- Support importing/exporting bookmarks from/to browsers (for importing existing bookmarks and for backup)
- Useful for people who hoard tabs and want to clean up their browser but still keep the links
- Tech stack ideas:
  - Bun for runtime and package manager (small footprint, fast startup)
  - React for frontend (Tanstack Start, Tailwind, Shadcn UI)
  - Sqlite for database (use Drizzle as ORM)
  - Something to handle background jobs for processing the links and generating tags (e.g. BullMQ, Agenda, or a custom solution using Bun's timers)
  - OpenRouter API for AI processing
  - PocketID for authentication
  - Docker for containerization and deployment

**Prompt for generating an app icon for the bookmarking service:**

```
Generate an app icon for an application that helps you organize bookmarks and links with the assistance of AI. The name of the application is 'bookmarkly'. Just create a graphical icon with no text. Use moss green as the main color and earth tones and beige tones for contrast colors. Don't add any rounded corners or masking, instead fill the whole space to the edges, but account for some space (approx 5%) will not be visible when masking will be applied by the system.
```

## An AI-powered image/icon generation studio

- Name ideas: **imager**, **artify**, **icongen**
- A web app where users can generate images using different AI models
- Support for multiple AI models (e.g. OpenAI GPT-5.4 Image 2, Stable Diffusion, DALL-E, Midjourney)
- Allow users to input prompts and generate images based on them
- Allow user to upload images as input for the AI models to modify or use as inspiration
- Implement a gallery where users can view and manage their generated images
- Use OpenRouter API to integrate with different AI models for image generation
- Implement user accounts and authentication using PocketID
- Tech stack ideas:
  - Bun for runtime and package manager
  - React for frontend (Tanstack Start, Tailwind, Shadcn UI)
  - Sqlite for database (use Drizzle as ORM)
  - OpenRouter API for AI processing
  - PocketID for authentication
  - Docker for containerization and deployment

## A Rust based CLI tool for renaming and organizing files using regex

- Name ideas: **renamr**, **regxr**, **orgnizr**
- A command-line tool that allows users to rename and organize files in a directory using regular expressions
- Support for batch renaming of files based on user-defined regex patterns
- Allow users to preview the changes before applying them
- Implement a dry-run mode to show what changes would be made without actually renaming the files
- Use Rust for performance and safety
- Use Clap for command-line argument parsing
- Use Regex crate for regex processing
- Use Walkdir for traversing directories
- Use Serde for configuration file parsing (if needed)
- Package the tool using Cargo and distribute it via crates.io
