# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Personal portfolio/profile website for Thuan Doan, hosted on GitHub Pages. Built with Jekyll using the Minima theme and the `github-pages` gem.

## Commands

```bash
# Install dependencies
bundle install

# Serve locally with live reload (http://localhost:4000)
bundle exec jekyll serve

# Build static site (output to _site/)
bundle exec jekyll build
```

Note: Changes to `_config.yml` require restarting the server.

## Architecture

- **Jekyll static site** using Minima 2.5 theme with `github-pages` gem
- **Custom layouts**: `base.html` → `page.html` (two-level layout chain; `page.html` wraps content in an `<article>` tag, `base.html` is the full HTML shell)
- **Content sections**:
  - `index.md` — homepage with bio, project gallery, and links
  - `projects/` — markdown pages for individual project write-ups
  - `research/` — markdown pages for research topics
  - `draft/` — CV and cover letter drafts (processed by Jekyll but not linked in nav)
  - `about.md` — about page
- **Navigation**: Custom `_includes/header.html` (nav links are currently commented out; site title links to home)
- **Static assets**: `assets/` contains images, CSS, and PDF resume
- **`_site/`**: Generated output directory (should not be edited directly)

## Content Conventions

- Project/research pages use `layout: page` front matter
- Inline `<style>` blocks are used in some markdown files for custom styling (e.g., removing table borders on the homepage)
- Images are referenced from `/assets/` with absolute paths
