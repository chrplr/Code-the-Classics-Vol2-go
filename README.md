# Go ports of the Arcade Games from _Code the Classics_

The books [Code the Classics, vol. I](https://magazine.raspberrypi.com/books/code-the-classics-vol-I-2ed)
and [Code the Classics, vol. II](https://magazine.raspberrypi.com/books/code-the-classics-vol-ii)
describe several arcade games programmed in Python with the pygame-zero library.

The original Python codes and the assets are provided by the authors at:

* vol. I — https://github.com/raspberrypipress/Code-the-Classics-Vol1
* vol. II — https://github.com/raspberrypipress/Code-the-Classics-Vol2

We translated the original Python codes to the [Go programming
language](http://go.dev), using the
[go-sdl3](https://github.com/Zyko0/go-sdl3) library.

We had two motivations:

- to check how go-sdl3 compares to python-pygame, for educational purposes
- to produce ready-to-run, self-contained binaries

## Play the games

A landing page with one button per game is available at:

* https://chrplr.github.io/Code-the-Classics-Vol1-go/
* https://chrplr.github.io/Code-the-Classics-Vol2-go/

The Go ports live in independent repositories:

| Game | Source | Play in browser |
|------|--------|-----------------|
| avenger-go | https://github.com/chrplr/avenger-go | https://chrplr.github.io/avenger-go/ |
| boing-go | https://github.com/chrplr/boing-go | https://chrplr.github.io/boing-go/ |
| myriapod-go | https://github.com/chrplr/myriapod-go | https://chrplr.github.io/myriapod-go/ |
| kinetix-go | https://github.com/chrplr/kinetix-go | https://chrplr.github.io/kinetix-go/ |
| cavern-go | https://github.com/chrplr/cavern-go | https://chrplr.github.io/cavern-go/ |
| eggzy-go | https://github.com/chrplr/eggzy-go | https://chrplr.github.io/eggzy-go/ |
| leadingedge-go | https://github.com/chrplr/leadingedge-go | https://chrplr.github.io/leading-edge-go/ |
| soccer-go | https://github.com/chrplr/soccer-go | https://chrplr.github.io/soccer-go/ |

Notes:
* A small library, [pgzgo](https://github.com/chrplr/pgzgo), was created to avoid duplication across the projects.
* Compilation relies (for now) on a fork of [go-sdl3](https://github.com/Zyko0/go-sdl3): https://github.com/chrplr/go-sdl3-wasm/tree/wasm-render-fixes

Christophe Pallier <christophe@pallier.org>  2026-07-06
