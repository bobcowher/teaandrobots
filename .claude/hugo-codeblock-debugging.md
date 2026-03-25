# Hugo Code Block Formatting Issue - Investigation Notes

## Problem
The replay buffer reference implementation post is rendering as a single blob of text instead of properly formatted code blocks. The code is not respecting newlines and appears without syntax highlighting.

## What We Discovered

### Root Cause
Hugo's goldmark markdown renderer is not converting fenced code blocks (` ```python ` or ` ``` `) into proper HTML `<pre><code>` structures. Instead, the code is being rendered as plain text without any HTML wrapper.

### Evidence
- Generated HTML shows code as plain text between paragraphs, not wrapped in `<pre><code>` tags
- Same issue occurs with both ````python` and plain ` ``` ` code blocks
- Theme uses PrismJS for syntax highlighting, which requires proper HTML structure from Hugo
- Creating new files from scratch shows identical behavior

### Theme Information
- Theme: hello-friend
- Uses PrismJS for syntax highlighting (not Hugo's built-in Chroma)
- Theme expects Hugo to generate proper `<pre><code>` HTML structures
- Theme example content uses plain ` ``` ` without language specifiers

### Configuration Attempts Made
1. Added modern goldmark configuration:
   ```toml
   [markup]
     [markup.goldmark]
       [markup.goldmark.renderer]
         unsafe = true
     [markup.highlight]
       codeFences = true
       guessSyntax = false
       noClasses = false
   ```

2. Tried legacy Pygments configuration:
   ```toml
   PygmentsCodeFences = true
   PygmentsStyle = "monokai"
   ```

3. Both approaches failed - Hugo still not generating `<pre><code>` structures

### Current Status
- Hugo build completes successfully
- All other markdown (paragraphs, headers) renders correctly
- Only fenced code blocks are affected
- Hugo version: v0.148.2-40c3d8233d4b123eff74725e5766fc6272f0a84d+extended

## Next Steps to Try
1. Revert to legacy Pygments configuration approach (was interrupted)
2. Check if there's a theme-specific configuration override
3. Examine Hugo's markdown processing pipeline in detail
4. Consider if this is a version compatibility issue between Hugo and the theme
5. Test with Hugo's server mode vs build mode
6. Check if there are any theme template overrides affecting code block rendering

## Files Modified
- `/content/posts/replay-buffer-reference-implementation.md` - changed ````python` to ```` ``` ````
- `/hugo.toml` - added various markup configurations

## Current Hugo Config
```toml
baseURL = 'https://www.teaandrobots.com/'
languageCode = 'en-us'
title = 'Tea & Robots'
theme = "hello-friend"

[markup]
  [markup.goldmark]
    [markup.goldmark.renderer]
      unsafe = true
  [markup.highlight]
    codeFences = true
    guessSyntax = false
    noClasses = false

[menu]
  [[menu.main]]
    identifier = "pages"
    name = "Pages"
    url = "/pages/"
    weight = 1
  [[menu.main]]
    identifier = "posts"
    name = "Posts"
    url = "/posts/"
    weight = 1
```