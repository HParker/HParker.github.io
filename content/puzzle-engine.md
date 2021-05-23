# puzzle engine

[Demo Video](https://www.youtube.com/watch?v=-mBRJC_mFdI) | [Code](https://github.com/hparker/puzzleengine) | [Inspiration](htpps://www.puzzlescript.net)

<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/-mBRJC_mFdI" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

_This video shows the PuzzleEngine test suite running through the tests with the optional renderer attached to show the puzzles being sovled at comically fast speeds._

PuzzleEngine is a PuzzleScript interpreter that allows you to run PuzzleScript games on your computer instead of in a web browser. You can find and play PuzzleScript games on [puzzlescript.net](htpps://www.puzzlescript.net).


## Technical Details

The parser uses `flex` and `bison` to lex and parse the program. The program is then converted accessed when rendering with whichever renderer you choose. I included a `SDL` based 2d renderer that is most feature complete as well as a quasi 3d renderer (using `raylib`) and an ASCII renderer (using `ncurses`).

### Example Program

PuzzleScript uses a declarative syntax to represent sokoban style block pushing games. A very simple PuzzleScript game might look like,

```
title Simple Block Pushing Game
author David Skinner
homepage www.puzzlescript.net

========
OBJECTS
========

Background
LIGHTGREEN GREEN
11111
01111
11101
11111
10111


Target
DarkBlue
.....
.000.
.0.0.
.000.
.....

Wall
BROWN DARKBROWN
00010
11111
01000
11111
00010

Player
Black Orange White Blue
.000.
.111.
22222
.333.
.3.3.

Crate
Orange Yellow
00000
0...0
0...0
0...0
00000


=======
LEGEND
=======

. = Background
# = Wall
P = Player
* = Crate
@ = Crate and Target
O = Target


=======
SOUNDS
=======

Crate MOVE 36772507

================
COLLISIONLAYERS
================

Background
Target
Player, Wall, Crate

======
RULES
======

[ >  Player | Crate ] -> [  >  Player | > Crate  ]

==============
WINCONDITIONS
==============

All Target on Crate

=======
LEVELS
=======


####..
#.O#..
#..###
#@P..#
#..*.#
#..###
####..


######
#....#
#.#P.#
#.*@.#
#.O@.#
#....#
######
```
