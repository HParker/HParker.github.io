This is my public TODO list. There really isn't a reason for it to be public, but here it is.

TODO:
- Write documentation on Ruby compiler instructions
- Write documentation on Ruby compiler instruction optimization flags
- Try static-rails
- emulators page
- finally get back to a gb/a emulator

piece tables:

"Hi there"

Original => "Hi there"
Add      => ""
Pieces   => [{ original 0-8 }]

"H there"

Original => "Hi there"
Add      => ""
Pieces   => [{ original 0-1, original 2-6 }]



"He there"

Original => "Hi there"
Add      => "e"
Pieces   => [{ original 0-1, add 0-1, original 2-6 }]


"Hey there"

Original => "Hi there"
Add      => "ey"
Pieces   => [{ original 0-1, add 0-2, original 2-6 }]
Pieces   => [{ original 0-1, add 0-1, add 1-1, original 2-6 }]
