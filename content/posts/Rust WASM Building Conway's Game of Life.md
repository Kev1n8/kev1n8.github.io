+++
title = "Rust WASM: Building Conway's Game of Life"
date = 2025-04-01

[taxonomies]
tags = ["rust", "wasm"]
+++

# Rust WASM: Building Conway's Game of Life

I've heard of wasm for quite a while, and finally decided to try it out.

This blog briefly writes the process of the practice.

## Prerequisites

Should know basic web development stuff, since wasm is run inside a browser (mostly).

- HTML
- CSS
- JavaScript

There are technically 2 ways of using rust wasm.

1. Build the whole project in Rust, or
2. Rust the heavy part of the project and use it as a lib.

Currently the latter one is preferred.

## Game of Life

The practice is going to focus on implementing Conway's Game of Life in Rust, building it to wasm and use it in JS.

The rules of the game are:

1. Any live cell with fewer than two live neighbours dies, as if by underpopulation.
2. Any live cell with two or three live neighbours lives on to the next generation.
3. Any live cell with more than three live neighbours dies, as if by overpopulation.
4. Any dead cell with exactly three live neighbours becomes a live cell, as if by reproduction.

For more info: https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life

## Implementation

### Rust Part

Rust comes with powerful tools like `wasm-pack`, `wasm-bindgen` and more for wasm programming.

We use `#[wasm-bindgen]` macro to make complier generates wasm-related files (js API file, wasm memory file, ts file and so on) when we use `wasm-pack build` to build our rust wasm lib.

I'll simply copy the code as follows:

```rust
use fixedbitset::FixedBitSet;
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
#[repr(u8)]
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Cell {
    Dead = 0,
    Alive = 1,
}

#[wasm_bindgen]
pub struct Universe {
    width: u32,
    height: u32,
    cells: FixedBitSet,
}

impl Universe {
    fn get_index(&self, row: u32, column: u32) -> usize {
        (row * self.width + column) as usize
    }

    fn live_neighbor_count(&self, row: u32, column: u32) -> u8 {
        let mut count = 0;
        for delta_row in [self.height - 1, 0, 1] {
            for delta_col in [self.width - 1, 0, 1] {
                if delta_row == 0 && delta_col == 0 {
                    continue;
                }

                let neighbor_row = (row + delta_row) % self.height;
                let neighbor_col = (column + delta_col) % self.width;
                let idx = self.get_index(neighbor_row, neighbor_col);
                count += self.cells[idx] as u8;
            }
        }
        count
    }
}

#[wasm_bindgen]
impl Universe {
    pub fn new() -> Self {
        let width = 64;
        let height = 64;

        let sz = (width * height) as usize;
        let mut cells = FixedBitSet::with_capacity(sz);

        for i in 0..sz {
            cells.set(i, js_sys::Math::random() < 0.5);
        }

        Self {
            width,
            height,
            cells,
        }
    }

    pub fn width(&self) -> u32 {
        self.width
    }

    pub fn height(&self) -> u32 {
        self.height
    }

    pub fn cells(&self) -> *const usize {
        self.cells.as_slice().as_ptr()
    }

    pub fn tick(&mut self) {
        let mut next = self.cells.clone();
        
        for row in 0..self.height {
            for col in 0..self.width {
                let idx = self.get_index(row, col);
                let cell = self.cells[idx];
                let live_neigbors = self.live_neighbor_count(row, col);

                next.set(idx, match (cell, live_neigbors) {
                    (true, x) if x < 2 => false,
                    (true, 2) | (true, 3) => true,
                    (true, x) if x > 3 => false,
                    (false, 3) => true,
                    (otherwise, _) => otherwise,
                });
            }
        }

        self.cells = next;
    }
}
```

### JavaScript Part

With the functional lib we built, now all we have to do is to use it to display the animation on the browser.

A basic web app is consist of:

- HTML: Defining the layout of the website
- CSS: Defining the style of the website
- JavaScript: Handling any user input and logical changes.

For HTML and CSS:

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Hello wasm-pack!</title>
  </head>
  <body>
    <canvas id="game-of-life-canvas"></canvas>
    <script src="./bootstrap.js"></script>
  </body>
  <style>
    body {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
    }
  </style>
</html>
```

For JS:

```JavaScript
import { Universe, Cell } from "wasm-game-of-life";
// Import the linear memory space of wasm.
import { memory } from "wasm-game-of-life/wasm_game_of_life_bg.wasm";

const CELL_SIZE = 5; // px
const GRID_COLOR = "#CCCCCC";
const DEAD_COLOR = "#FFFFFF";
const ALIVE_COLOR = "#000000";

// Construct the universe, and get its width and height.
const universe = Universe.new();
const width = universe.width();
const height = universe.height();

// Give the canvas room for all of our cells and a 1px border
// around each of them.
const canvas = document.getElementById("game-of-life-canvas");
canvas.height = (CELL_SIZE + 1) * height + 1;
canvas.width = (CELL_SIZE + 1) * width + 1;

const ctx = canvas.getContext('2d');

const renderLoop = () => {
    universe.tick();

    drawGrid();
    drawCells();

    requestAnimationFrame(renderLoop);
};

const drawGrid = () => {
    ctx.beginPath();
    ctx.strokeStyle = GRID_COLOR;

    // Vertical lines.
    for (let i = 0; i <= width; i++) {
        ctx.moveTo(i * (CELL_SIZE + 1) + 1, 0);
        ctx.lineTo(i * (CELL_SIZE + 1) + 1, (CELL_SIZE + 1) * height + 1);
    }

    // Horizontal lines.
    for (let j = 0; j <= height; j++) {
        ctx.moveTo(0, j * (CELL_SIZE + 1) + 1);
        ctx.lineTo((CELL_SIZE + 1) * width + 1, j * (CELL_SIZE + 1) + 1);
    }

    ctx.stroke();
};

const getIndex = (row, column) => {
    return row * width + column;
};

const drawCells = () => {
    // Accessing the buffer of wasm.
    const cellsPtr = universe.cells();
    const cells = new Uint8Array(memory.buffer, cellsPtr, width * height / 8);

    ctx.beginPath();

    for (let row = 0; row < height; row++) {
        for (let col = 0; col < width; col++) {
            const idx = getIndex(row, col);

            ctx.fillStyle = cells[idx] === Cell.Dead
                ? DEAD_COLOR
                : ALIVE_COLOR;

            ctx.fillRect(
                col * (CELL_SIZE + 1) + 1,
                row * (CELL_SIZE + 1) + 1,
                CELL_SIZE,
                CELL_SIZE
            );
        }
    }

    ctx.stroke();
};

drawGrid();
drawCells();
requestAnimationFrame(renderLoop);
```

## Testing

Example test.rs:

```rust
// Only test for target: "wasm-unknown-unknown"
#![cfg(target_arch = "wasm32")]
#![cfg(test)]

// Use wasm_bindgen_test and our [`Universe`];
use wasm_bindgen_test::*;
use wasm_game_of_life::Universe;

// Configure our test to run in the browser.
wasm_bindgen_test_configure!(run_in_browser);

pub fn input_spaceship() -> Universe {
    let mut universe = Universe::new();
    universe.set_width(6);
    universe.set_height(6);
    universe.set_cells(&[(1,2), (2,3), (3,1), (3,2), (3,3)]);
    universe
}

pub fn expected_spaceship() -> Universe {
    let mut universe = Universe::new();
    universe.set_width(6);
    universe.set_height(6);
    universe.set_cells(&[(2,1), (2,3), (3,2), (3,3), (4,2)]);
    universe
}

// Similar to #[test]
#[wasm_bindgen_test]
pub fn test_tick() {
    let mut input_universe = input_spaceship();
    let expected_universe = expected_spaceship();

    input_universe.tick();
    assert_eq!(input_universe.get_cells(), expected_universe.get_cells());
}
```
By setting the test rs file, we can simply `wasm-pack test --chrome --headless` to test our test suite.

`--chrome` means we prefer to run the test in Google Chrome, so be sure the latest Google Chrome is downloaded.

`--headless` means we prefer to test without opening the browser and viewing.

## Debugging

### Log for Panics

If our rust wasm panics for unknown reasons, we would like to at least be aware of the location of panicking.

We could achieve this by using the crate `console_error_panic_hook`.

Usage is simply:

```rust
// Place this in the init function.
// `set_once` ensure `set_hook` is called once.
console_error_panic_hook::set_once();
```

Then, if the rust part panics, we'll get messages like (build with `wasm-pack build --dev`):
```
panicked at src/lib.rs:65:9:
explicit panic

Stack:

@
logError@
wasm_game_of_life.wasm.wasm-function[__wbg_new_8a6f238a6ece86ea externref shim]@[wasm code]
wasm_game_of_life.wasm.wasm-function[console_error_panic_hook::Error::new::h19e56b4cb73b36b1]@[wasm code]
wasm_game_of_life.wasm.wasm-function[console_error_panic_hook::hook_impl::hbfe233a683db229c]@[wasm code]
wasm_game_of_life.wasm.wasm-function[console_error_panic_hook::hook::h9ea87a8daf4cfd59]@[wasm code]
wasm_game_of_life.wasm.wasm-function[core::ops::function::Fn::call::h9d1dc855776bf28e]@[wasm code]
wasm_game_of_life.wasm.wasm-function[std::panicking::rust_panic_with_hook::h9ea01914b5d92659]@[wasm code]
wasm_game_of_life.wasm.wasm-function[std::panicking::begin_panic::{{closure}}::h1965b49f537d1c88]@[wasm code]
wasm_game_of_life.wasm.wasm-function[std::sys::backtrace::__rust_end_short_backtrace::hdeabdd4295ec7969]@[wasm code]
wasm_game_of_life.wasm.wasm-function[std::panicking::begin_panic::hbacf3f38466eeebe]@[wasm code]
wasm_game_of_life.wasm.wasm-function[wasm_game_of_life::Universe::new::hc5d33d50da932d85]@[wasm code]
wasm_game_of_life.wasm.wasm-function[universe_new]@[wasm code]
new@
@
```

### Log Error Messages to Console

It's useful to log info if something went wrong. How do we log then?

We need another dependency, `web-sys`, which provides us the `console::log` API.

Well, simple as that. Then we can write a macro to easy up the coding:

```rust
macro_rules! log {
    ( $( $t:tt )* ) => {
        web_sys::console::log_1(&format!( $($t)* ).into());
    };
}

// Log while ticking.
log!(
    "cell[{}, {}] is initially {} and has {} live neighbors",
    row,
    col,
    cell,
    live_neigbors
);
```

### Use a Debugger

Either by placing an `debugger` in advanced in `.js` or by clicking in the sources interface of the browser is ok for js.

However, there's currently no mature debugging tool for rust wasm. We may end up stepping into raw wasm instructions rather than the Rust source text we authored.

## Interaction between Rust and JS

We are going to add 2 interactions:

1. Play and pause button, and
2. Toggle cell state by clicking

### Add a Button

Add a button in `.html` with `<button id="play-pause"></button>`.

In `index.js`, add the logic code:

```javascript
const playPauseButton = document.getElementById("play-pause");

const isPaused = () => {
    return animationId === null;
};

const play = () => {
    playPauseButton.textContent = "⏸";
    renderLoop();
}

const pause = () => {
    playPauseButton.textContent = "▶";
    cancelAnimationFrame(animationId);
    animationId = null; // This would be set in `renderLoop`.
};

playPauseButton.addEventListener("click", event => {
    if (isPaused()) {
        play();
    } else {
        pause();
    }
});
```

### Toggling Cells

First we have to impl the toggle method on the rust side.

```rust
impl Universe {
    pub fn toggle_cell(&mut self, row: u32, col: u32) {
        let idx = self.get_index(row, col);
        self.cells.set(idx, !self.cells[idx]);
    }
}
```

Then, we add the toggling logic code in js.

```javascript
canvas.addEventListener("click", event => {
    const boundingRect = canvas.getBoundingClientRect();

    const scaleX = canvas.width / boundingRect.width;
    const scaleY = canvas.height / boundingRect.height;

    const canvasLeft = (event.clientX - boundingRect.left) * scaleX;
    const canvasTop = (event.clientY - boundingRect.top) * scaleY;

    const row = Math.min(Math.floor(canvasTop / (CELL_SIZE + 1)), height - 1);
    const col = Math.min(Math.floor(canvasLeft / (CELL_SIZE + 1), width - 1));

    universe.toggle_cell(row, col);

    drawCells();
})
```

## Time Profiling

### Counting FPS by `performance.now()`

Add dependency `web-sys` with features `["Window", "Performance"]` if would like to count fps in Rust.

Or we can simply count it in js:

```javascript

const fps = new class {
    constructor() {
        this.fps = document.getElementById("fps");
        this.frames = [];
        this.lastFrameTimeStamp = performance.now();
    }

    render() {
        const now = performance.now();
        const delta = now - this.lastFrameTimeStamp;
        this.lastFrameTimeStamp = now;
        const fps = 1 / delta * 1000;

        this.frames.push(fps);
        if (this.frames.length > 100) {
            this.frames.shift();
        }

        let min = Infinity;
        let max = -Infinity;
        let sum = 0;
        for (let i = 0; i < this.frames.length; i++) {
            sum += this.frames[i];
            min = Math.min(this.frames[i], min);
            max = Math.max(this.frames[i], max);
        }
        let mean = sum / this.frames.length;

        this.fps.textContent = `
Frames per Second:
         latest = ${Math.round(fps)}
avg of last 100 = ${Math.round(mean)}
min of last 100 = ${Math.round(min)}
max of last 100 = ${Math.round(max)}
`.trim();
    }
};
```
And call `fps.render()` in `renderLoop`.

### Set Timer for `Universe::tick`

```rust
pub struct Timer<'a> {
    name: &'a str,
}

impl<'a> Timer<'a> {
    pub fn new(name: &'a str) -> Self {
        console::time_with_label(name);
        Self { name }
    }
}

impl<'a> Drop for Timer<'a> {
    fn drop(&mut self) {
        console::time_end_with_label(self.name);
    }
}
```

By that the timer ticking will show outputs in the js console.

(TODO: Does it brings more info in the recorded frames(waterfalls)?)

### Enhancing Performance

With the timer above, we can inspect the time cost of each function calls.

For some reason my bottleneck is different from the project guide, but I'll first enhance the render part to keep the same with the guide anyway, which is to skip `fillRect` when it's not necessary.

The bottleneck of my implementation is `Universe::tick`. Let's add more timer in between:

```rust
pub fn tick(&mut self, times: u32) {
        let mut next = {
            let _timer = Timer::new("allocate next cells");
            self.cells.clone()
        };

        for _ in 0..times {
            let _timer = Timer::new("new_genaration");
            for row in 0..self.height {
                for col in 0..self.width {
                    let idx = self.get_index(row, col);
                    let cell = self.cells[idx];
                    let live_neigbors = self.live_neighbor_count(row, col);

                    let next_cell = match (cell, live_neigbors) {
                        (true, x) if x < 2 => false,
                        (true, 2) | (true, 3) => true,
                        (true, x) if x > 3 => false,
                        (false, 3) => true,
                        (otherwise, _) => otherwise,
                    };
                    next.set(idx, next_cell);
                }
            }
        }

        let _timer = Timer::new("free old cells");
        self.cells = next;
    }
```

TODO: for some reason I don't see the timer with these labels in the dev tools.

Any way, according to the flamegraph, the most time consuming function call is `Universe::live_neighbor_count` (check out the orange bar in the pic below).

![Flamegraph showing the time-consuming function call in Universe::live_neighbor_count](./Task%20Archive/images/wasm%20count%20neighbor%20flamegraph.png)

## Shrinking Binary Size

Use `wasm-opt` tool to shrink code size:

```bash
# Optimize for size.
wasm-opt -Os -o output.wasm input.wasm

# Optimize aggressively for size.
wasm-opt -Oz -o output.wasm input.wasm

# Optimize for speed.
wasm-opt -O -o output.wasm input.wasm

# Optimize aggressively for speed.
wasm-opt -O3 -o output.wasm input.wasm
```

For further optimizations, we could turn our `Universe` into a static global instance, and so is its cells. Without any allocator, we could even make our lib `#![no_std]`

I'll leave it here, though.

## Publishing

Publish to npm:

```bash
wasm-pack login
wasm-pack build
wasm-pack publish
```
