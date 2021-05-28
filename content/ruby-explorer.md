# Ruby Explorer

[GitHub](https://github.com/HParker/ruby_explorer) | [Site](https://www.rubyexplorer.xyz/)

![Ruby Explorer Screenshot](/ruby_explorer.png)

Ruby Explorer makes it easy to explore and share Ruby VM internals. Typing Ruby code into Ruby Explorer generates a list of VM instructions created by ruby with links to their implementation in Ruby's source code. In addition you can see what tokens the lexer returned and the S-Expression that ruby generated from those tokens.

I became interested in making this tool while reading [Ruby Under A Microscope](https://nostarch.com/rum) by [AUTHOR](http://patshaughnessy.net/). I could run `ripper` to get the parse data or ruby `ruby --dump=insns` and get the instructions, but having a tool that does all that **and** links me the definition in ruby source seemed super worth having.
