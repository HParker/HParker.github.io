# Uncovering Ruby Bytecode Patterns

![Cute Stack Machine](/cute-stack-machine.jpg)

Since Ruby 1.9, Ruby runs your code in a bytecode VM. That means that the ruby compiler converts your code to a series of bytecode instructions. For example,

```text
ruby --dump=insns -e '5 * 10'

== disasm: #<ISeq:<main>@-e:1 (1,0)-(1,6)> (catch: false)

0000 putobject                              5

0002 putobject                              10

0004 opt_mult                               <calldata!mid:*, argc:1, ARGS_SIMPLE>[CcCr]

0006 leave
```

The bytecode instructions `putobject` is called twice, `opt_mult` is called next, then lastly `leave`. These are the bytecode instructions that Ruby runs when executing `5 * 10`. Ruby uses a stack based VM, so after `putobject 5` is called, `5` is on the stack to be used by other instructions.

**What instructions the Ruby VM is actually running?** Finding common patterns could lead to interesting optimizations and a better understanding of the Ruby VM. Ruby provides hooks for `dtrace`/`systemtap` to give this information. a DTrace probe can send us information every time an instruction is run in the VM.

## Setup

build Ruby with DTrace,

```bash
git clone https://github.com/ruby/ruby.git
./autogen.sh
mkdir -p ~/.rubies
./configure --prefix="${HOME}/.rubies/ruby-master" --enable-dtrace
```

In `vm_opts.h` set `VM_COLLECT_USAGE_DETAILS` to 1

then,

```
make install
```

Create this stp script somewhere handy,

```
// ruby-instructions.stp
probe process("/path/to/ruby/bin/ruby").mark("insn")
{
    printf("%s\n", user_string($arg1))
}
```

Setup rails bench benchmark from https://github.com/k0kubun/railsbench

Make sure to run single threaded and copy the pid from the web worker for the next step.

Capture with,

```
bundle exec puma -e production --threads=1
sudo stap ruby-instructions.stp -o rails-bench.txt -x <web worker pid>
ab -c 1 -n 10000 localhost:3000/posts
```

You should now have a text file that is a long list of instructions Ruby ran during your test.

[Full Parsed Results](https://gist.github.com/HParker/65c9ada9614301526a182b58be4f86fd)

## What instructions are most common?

![Instruction Chart](/instruction-chart.png)

* getlocal_WC_0 = 1,903,656
* opt_send_without_block = 1,638,356
* leave = 1,057,370
* putself = 729,499
* branchunless = 626,085
* putobject = 584,513
* setlocal_WC_0 = 502,096
* getinstancevariable = 434,992
* pop = 407,503
* dup = 375,151

It seems `getlocal_WC_0`, `opt_send_without_block`, `leave` and `putself` are the most common instructions run in rails benchmark.

## What instruction pairs are common?

After an instruction ran, what instructions are most likely to follow it? Because we are measuring a running program, these instructions are not necessarily generated next to each other, but instead they must be executed one after another.

### getlocal_WC_0

![getlocal_WC_0-chart](/getlocal_WC_0-chart.png)

`getlocal_WC_0` is very often followed by `opt_send_without_block` or `getlocal_WC_0` calling itself again. Often running this instruction twice could mean that local access for multiple variables being as fast as possible could show good results in Rails bench.

### opt_send_without_block

![opt_send_without_block-chart](/opt_send_without_block-chart.png)

### leave

![leave-chart](/leave-chart.png)

The pattern `leave` -> `leave` is interesting. It is possible Ruby generates bytecode where `leave` is always called twice in a row. Unfortunately, it might be very uncommon that Ruby can know that two `leave` instructions are always called one after another. I am curious if Ruby could identify these situations.

### putself

![putself-chart](/putself-chart.png)

This pattern seems like a, "get ready for `opt_send_without_block`" pattern that `putself` and `getlocal` are both a part of.

### branchunless

![branchunless-chart](/branchunless-chart.png)

### putobject

![putobject-chart](/putobject-chart.png)

### setlocal_WC_0

![setlocal_WC_0-chart](/setlocal_wc_0-chart.png)

I am really curious how commonly these `setlocal` -> `getlocal` pattern is referencing the same variable. This could be replaced with `dup` -> `setlocal`, though that is only useful if these references are to the same variable. Alternatively, a "stack preserving" version of setlocal might be a different interesting alternative.

### getinstancevariable

![getinstancevariable-chart](/getinstancevariable-chart.png)

### pop

![pop-chart](/pop-chart.png)

### dup

![dup-chart](/dup-chart.png)

A common pattern starting with `dup` is, `["dup", "branchif", "pop"]`. This pattern ran 912,485 in our test. This instruction sequence gets generated from `x ||= 1`. [rubyexplorer.xyz](http://www.rubyexplorer.xyz/explores?code=x+%7C%7C%3D+1&coverage_enabled=true&debug_frozen_string_literal=false&frozen_string_literal=false&inline_const_cache=true&instructions_unification=false&operands_unification=true&peephole_optimization=true&specialized_instruction=true&stack_caching=false&tailcall_optimization=false)

```text
== disasm: #<ISeq:<compiled>@<compiled>:1 (1,0)-(1,7)> (catch: FALSE)
local table (size: 1, argc: 0 [opts: 0, rest: -1, post: 0, block: -1, kw: -1@-1, kwrest: -1])
[ 1] x@0
0000 getlocal_WC_0 x@0 ( 1)[Li]
0002 dup
0003 branchif 10
0005 pop
0006 putobject_INT2FIX_1_
0007 dup
0008 setlocal_WC_0 x@0
0010 leave
```

This instruction sequence duplicates the top element on the stack, branches if that element is truthy and then removes the element it duplicated from the stack. If we are going to remove the original element on the stack anyways, why duplicate it in the first place? Turns out the Ruby VM knows that this sequence is suboptimal and has the information to generate a better sequence. This is the change to generate that better sequence: [ruby/ruby#6414](https://github.com/ruby/ruby/pull/6414). This optimization only has any effect if Ruby knows that the result of the conditional assignment is unused. After this change, conditional assignment where the result is ignored is *1.72x faster*.

### objtostring

![objtostring-chart](/objtostring-chart.png)

`objtostring` is almost always followed by `anytostring`. These instructions are used nearly exclusively for string interpolation [rubyexplorer.xyz](https://www.rubyexplorer.xyz/explores?code=%22%23%7B123%7D%22&coverage_enabled=true&debug_frozen_string_literal=false&frozen_string_literal=false&inline_const_cache=true&instructions_unification=false&operands_unification=true&peephole_optimization=true&specialized_instruction=true&stack_caching=false&tailcall_optimization=false). There have been two interesting Ruby changes related to string interpolation [ruby/ruby#6334](https://github.com/ruby/ruby/pull/6334) and [ruby/ruby#6335](https://github.com/ruby/ruby/pull/6335)

#### P.S.

While writing this I rediscovered [Tenderlove's old post about introducing Dtrace](http://tenderlovemaking.com/2011/12/05/profiling-rails-startup-with-dtrace.html). A good read if you find the time.

Also, if you are interested in learning more about YARV and the Ruby VM. I recommend [this YARV reference](https://kddnewton.com/yarv/) and [Ruby Under a Microscope](https://nostarch.com/rum).