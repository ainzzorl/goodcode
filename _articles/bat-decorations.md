---
title:  "Bat - Text Decoration [Rust]"
layout: default
last_modified_date: 2021-08-01T18:34:00+0300
nav_order: 8

status: PUBLISHED
language: Rust
short-title: Text Decoration
project:
  name: Bat
  key: bat
  home-page: https://github.com/sharkdp/bat
tags: [cli, decorator]
---

{% include article-meta.html article=page %}

## Context

Bat is a *cat(1)* clone with syntax highlighting and Git integration.

## Problem

Bat displays texts with "decorations": line numbers, change indicator, grid border. These decorations can be used in any combination depending on the user input.

## Overview

Text printing is done by [`InteractivePrinter`](https://github.com/sharkdp/bat/blob/375d55aa5d7f3390e33febcc40a8d629b22926ae/src/printer.rs#L102-L114). [`InteractivePrinter`](https://github.com/sharkdp/bat/blob/375d55aa5d7f3390e33febcc40a8d629b22926ae/src/printer.rs#L102-L114) maintains a list of [`Decoration`](https://github.com/sharkdp/bat/blob/375d55aa5d7f3390e33febcc40a8d629b22926ae/src/decorations.rs#L12-L20)s and populates it based on user config.

[`Decoration`](https://github.com/sharkdp/bat/blob/375d55aa5d7f3390e33febcc40a8d629b22926ae/src/decorations.rs#L12-L20) trait has a method `generate` accepting `line_number`, `continuation` (if the line is being broken into shorter lines) and [`InteractivePrinter`](https://github.com/sharkdp/bat/blob/375d55aa5d7f3390e33febcc40a8d629b22926ae/src/printer.rs#L102-L114) and returning [`DecorationText`](https://github.com/sharkdp/bat/blob/375d55aa5d7f3390e33febcc40a8d629b22926ae/src/decorations.rs#L6-L10). The printer then prints all enabled decorations before printing the line content.

## Implementation details

[`Decoration` trait](https://github.com/sharkdp/bat/blob/375d55aa5d7f3390e33febcc40a8d629b22926ae/src/decorations.rs#L12-L20):
```rust
pub(crate) trait Decoration {
    fn generate(
        &self,
        line_number: usize,
        continuation: bool,
        printer: &InteractivePrinter,
    ) -> DecorationText;
    fn width(&self) -> usize;
}
```

It resembles the [Decorator pattern](https://en.wikipedia.org/wiki/Decorator_pattern), but it's not quite the same. The classical Decorator wraps the original class to augment its behavior without changing its interface.

[One of the decorations](https://github.com/sharkdp/bat/blob/375d55aa5d7f3390e33febcc40a8d629b22926ae/src/decorations.rs#L22-L70):
```rust
pub(crate) struct LineNumberDecoration {
    color: Style,
    cached_wrap: DecorationText,
    cached_wrap_invalid_at: usize,
}

impl LineNumberDecoration {
    pub(crate) fn new(colors: &Colors) -> Self {
        LineNumberDecoration {
            color: colors.line_number,
            cached_wrap_invalid_at: 10000,
            cached_wrap: DecorationText {
                text: colors.line_number.paint(" ".repeat(4)).to_string(),
                width: 4,
            },
        }
    }
}

impl Decoration for LineNumberDecoration {
    fn generate(
        &self,
        line_number: usize,
        continuation: bool,
        _printer: &InteractivePrinter,
    ) -> DecorationText {
        if continuation {
            if line_number > self.cached_wrap_invalid_at {
                let new_width = self.cached_wrap.width + 1;
                return DecorationText {
                    text: self.color.paint(" ".repeat(new_width)).to_string(),
                    width: new_width,
                };
            }

            self.cached_wrap.clone()
        } else {
            let plain: String = format!("{:4}", line_number);
            DecorationText {
                width: plain.len(),
                text: self.color.paint(plain).to_string(),
            }
        }
    }

    fn width(&self) -> usize {
        4
    }
}
```

[Instantiating decorations](https://github.com/sharkdp/bat/blob/375d55aa5d7f3390e33febcc40a8d629b22926ae/src/printer.rs#L133-L164):

```rust
// Create decorations.
let mut decorations: Vec<Box<dyn Decoration>> = Vec::new();

if config.style_components.numbers() {
    decorations.push(Box::new(LineNumberDecoration::new(&colors)));
}

#[cfg(feature = "git")]
{
    if config.style_components.changes() {
        decorations.push(Box::new(LineChangesDecoration::new(&colors)));
    }
}

let mut panel_width: usize =
    decorations.len() + decorations.iter().fold(0, |a, x| a + x.width());

// The grid border decoration isn't added until after the panel_width calculation, since the
// print_horizontal_line, print_header, and print_footer functions all assume the panel
// width is without the grid border.
if config.style_components.grid() && !decorations.is_empty() {
    decorations.push(Box::new(GridBorderDecoration::new(&colors)));
}

// Disable the panel if the terminal is too small (i.e. can't fit 5 characters with the
// panel showing).
if config.term_width
    < (decorations.len() + decorations.iter().fold(0, |a, x| a + x.width())) + 5
{
    decorations.clear();
    panel_width = 0;
}
```

[Applying decorations in `InteractivePrinter`](https://github.com/sharkdp/bat/blob/375d55aa5d7f3390e33febcc40a8d629b22926ae/src/printer.rs#L412-L424):
```rust
// Line decorations.
if self.panel_width > 0 {
    let decorations = self
        .decorations
        .iter()
        .map(|ref d| d.generate(line_number, false, self))
        .collect::<Vec<_>>();

    for deco in decorations {
        write!(handle, "{} ", deco.text)?;
        cursor_max -= deco.width + 1;
    }
}
```

## Testing

It's tested with "snapshot tests". E.g. [this test input](https://github.com/sharkdp/bat/blob/master/tests/snapshots/sample.rs) is expected to be rendered to [this](https://github.com/sharkdp/bat/blob/master/tests/snapshots/output/numbers.snapshot.txt) when line number decoration is enabled.

## References

* [GitHub repo](https://github.com/sharkdp/bat)

## Copyright notice

Bat is licensed under the [Apache License 2.0](https://github.com/sharkdp/bat/blob/master/LICENSE-APACHE).

Copyright (c) 2018-2021 bat-developers.
