# Go ports of the Arcade Games from _Code the Classics Volume II_

The book [Code the Classics, vol.2](https://magazine.raspberrypi.com/books/code-the-classics-vol-ii) describes several arcage games programmed using Python and the pygame-zero library. 

The original codes and the assets are provided at https://github.com/raspberrypipress/Code-the-Classics-Vol2

We translated, the original Python codes to the [Go programming language](http://go.dev), using the [go-sdl3](https://github.com/Zyko0/go-sdl3). 

We had two motivations:

- to check how go-sdl3 compares to python-pygame, for educational purposes
- to produce ready-to-run, self-contained binaries

The Go ports are in independent repositories:

The Go ports are in independent repositories:

* avenger-go	https://github.com/chrplr/avenger-go  ([play it!](https://chrplr.github.io/avenger-go/))
* boing-go	https://github.com/chrplr/boing-go ([play it!](https://chrplr.github.io/boing-go/))
* myriapod-go	https://github.com/chrplr/myriapod-go ([play it!](https://chrplr.github.io/myriapod-go/))
* kinetix-go	https://github.com/chrplr/kinetix-go ([play it!](https://chrplr.github.io/kinetix-go/))
* cavern-go	https://github.com/chrplr/cavern-go ([play it!](https://chrplr.github.io/cavern-go/))
* eggzy-go	https://github.com/chrplr/eggzy-go ([play it!](https://chrplr.github.io/eggzy-go/))
* leadingedge-go	https://github.com/chrplr/leadingedge-go ([play it!](https://chrplr.github.io/leading-edge-go/))
* soccer-go	https://github.com/chrplr/soccer-go ([play it!](https://chrplr.github.io/soccer-go/))

Notes:
* A small library, [pgzgo](http://github.com:chrplr/pgzgo) was created to avoid duplication across the projects.
* Compilation relies (for now) on a fork of [go-sdl3](https://github.com/Zyko0/go-sdl3): https://github.com/chrplr/go-sdl3-wasm/tree/wasm-render-fixes

Christophe Pallier <christophe@pallier.org>  2026-07-06

