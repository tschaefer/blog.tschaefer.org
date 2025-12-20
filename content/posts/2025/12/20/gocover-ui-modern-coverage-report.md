---
title: "gocover-ui - Modern HTML Coverage Report"
description: "Transforms Go coverage reports into interactive, visual
representations that make it easier to understand and act upon the data."
author: "Tobias Schäfer"
date: 2025-12-20T19:01:48+01:00
draft: false
toc: false
images:
    - "https://blog.tschaefer.org/images/gocover-ui-index.png"
    - "https://blog.tschaefer.org/images/gocover-ui-file.png"
tags:
  - go
  - coverage
  - testing
  - gocover-ui
---

When it comes to software development, having clear insights into your code's
test coverage is invaluable. Coverage data informs where improvements are
needed and helps maintain high-quality standards. Go offers built-in tools for
generating coverage reports, but these reports can often be difficult to
interpret.

![gocover-ui Index view](/images/gocover-ui-index.png)

I created **gocover-ui** to address this challenge and provide an alternative
to cloud based coverage reporting tools, focusing on local, interactive HTML
reports.

## Overview

- **Index View**: A comprehensive overview with a file browser and a donut
  graph that visualizes overall coverage.
- **File View**: Detailed line-by-line coverage information with color-coded
  highlights for covered, partial covered, and missed lines.

## Index View

The file browser provides a hierarchical representation of your codebase. It
allows you to navigate through nodes - directories and files, making it easy to
find specific areas of interest. Any node provides statistics about total,
covered, partial covered, missed lines and coverage percentage, colored
according to the coverage band.

The donut graph visualizes the overall coverage percentage of the selected
hierachy level. The file browser nodes are represented as arcs. The
arc length is proportional to the tracked lines and the segment color reflects
the node coverage band.

The coverage band colors are in the range from red over yellow to green,
indicating low to high coverage.

## File View

A click on a donut graph arc representing a file or a file node in the browser
opens the file view. It displays the source code with line-by-line coverage
information. Covered lines are marked in green, partially covered lines in
yellow, and missed lines in red. This color-coding makes it easy to identify
areas that need attention. Lines can be highlighted by a click on the
respective line number.

![gocover-ui File view](/images/gocover-ui-file.png)

## Installation and Usage

- Fetch the latest version of **gocover-ui** from the [GitHub repository releases
page](https://github.com/tschaefer/gocover-ui/releases).

- Generate a coverage profile using Go's built-in testing tools:
```bash
go test -coverprofile=coverage.out ./...
```

- Run **gocover-ui** with the generated coverage profile:
```bash
gocover-ui -coverprofile=coverage.out -src .
```

- Open the generated HTML report in your web browser to explore the coverage
  data.

That's it! You now have a powerful tool to visualize and analyze your Go code
coverage.

## Conclusion

**gocover-ui** transforms Go coverage reports into interactive, visual
representations that make it easier to understand and act upon the data. By
providing both an index view and a detailed file view, it empowers to quickly
identify areas for improvement and maintain high-quality codebases.

Give **gocover-ui** a try in your next Go project and experience the benefits
of enhanced coverage reporting!
