---
title: Hello hello world
draft: true
---

TODO: maybe only reference 0.6.0 docs as they might not be as ephemeral and also

Languages are often judged initially on their "[Hello,
world!](https://en.wikipedia.org/wiki/%22Hello,_World!%22_program)" program.
How easy is it to write? To run? How easy is it to understand? It's a very
simple program, of course, one of the simplest, even... just produce a little
text, and display it, what could be simpler?

It's really not fair to judge a language by such a cursory impression, but it
_can_ give you an idea of what a language _values_ and how it works. What does
the syntax look like? Is it typed? Is it interpreted? You can usually tell a
lot at a glance.

For example, One of Ruby's (many) hello worlds is so simple, it's also Python!

```ruby
print('Hello world!')
```

Often, people coming from interpreted languages experience compiled, systems
languages to be more complicated right off the bat. There is the obvious added
complexity of compiling and running as separate steps, as opposed to simply
pointing an executable at some source code and seeing a result right away, but
there are often syntactical constructs to go along with that..

At first glance, Rust's hello world looks fairly inert, too:

```rust
fn main() {
    println!("Hello World!");
}
```

But `println!` is actually a macro, what does it look like expanded?

```rust
macro_rules! println {
    () => ($crate::print!("\n"));
    ($($arg:tt)*) => ({
        $crate::io::_print($crate::format_args_nl!($($arg)*));
    })
}
```

You'll notice that this _also_ has a macro _inside of it._ This calls into `print!` which expands to call into

```rust
macro_rules! print {
    ($($arg:tt)*) => ($crate::io::_print($crate::format_args!($($arg)*)));
}
```

`io::_print` which calls into

```rust
pub fn _print(args: fmt::Arguments<'_>) {
    print_to(args, &LOCAL_STDOUT, stdout, "stdout");
}
```

`print_to` which looks like

```rust
fn print_to<T>(
    args: fmt::Arguments<'_>,
    local_s: &'static LocalKey<RefCell<Option<Box<dyn Write + Send>>>>,
    global_s: fn() -> T,
    label: &str,
) where
    T: Write,
{
    let result = local_s
        .try_with(|s| {
            // Note that we completely remove a local sink to write to in case
            // our printing recursively panics/prints, so the recursive
            // panic/print goes to the global sink instead of our local sink.
            let prev = s.borrow_mut().take();
            if let Some(mut w) = prev {
                let result = w.write_fmt(args);
                *s.borrow_mut() = Some(w);
                return result;
            }
            global_s().write_fmt(args)
        })
        .unwrap_or_else(|_| global_s().write_fmt(args));

    if let Err(e) = result {
        panic!("failed printing to {}: {}", label, e);
    }
}

```

Which is, uh, well let's just say it's not exactly clear what's happening at a
glance. There is a lot going on here!

Now, to be clear, I'm not faulting Rust here at all, my point is exactly the
opposite actually, in that there is _always_ necessarily more going on in a
hello world than `puts "la da da"` or similar would have you believe on its
face. Speaking of Ruby's `puts`, what _is_ the code that runs `puts` in the
Ruby interpreter itself, which is written in C?

Well it looks like [this](https://github.com/ruby/ruby/blob/7c2bbd1c7d40a30583844d649045824161772e36/io.c#L7727-L7758)

```c
VALUE
rb_io_puts(int argc, const VALUE *argv, VALUE out)
{
    int i, n;
    VALUE line, args[2];

    /* if no argument given, print newline. */
    if (argc == 0) {
        rb_io_write(out, rb_default_rs);
        return Qnil;
    }
    for (i=0; i<argc; i++) {
        if (RB_TYPE_P(argv[i], T_STRING)) {
            line = argv[i];
            goto string;
        }
        if (rb_exec_recursive(io_puts_ary, argv[i], out)) {
            continue;
        }
        line = rb_obj_as_string(argv[i]);
      string:
        n = 0;
        args[n++] = line;
        if (RSTRING_LEN(line) == 0 ||
            !rb_str_end_with_asciichar(line, '\n')) {
            args[n++] = rb_default_rs;
        }
        rb_io_writev(out, n, args);
    }

    return Qnil;
}
```

Hello world!

We all know that a languages like Ruby or Python are designed explicitly to
hide this sort of complexity from us and let us get on with the dirty business
of munging data blobs or serving web requests or whatever, and thank goodness
for that, but wow that _is_ a lot, isn't it?

---

I think when people come from languages that were designed to be ergonomic to
more systems oriented languages, they're often jarred by what they perceive to
be inelegant, ugly, and verbose code. And to be sure, it _is_ sometimes exactly
that, but usually, the tradeoff is explicit: elegance and simplicity for
_control_...  for specific, granular control over the program that is
eventually run.  It isn't always necessary, in fact almost always UNnecessary,
to have _that_ much control over your program. Obviously, productivity matters,
and if your business is *_insert business_*, well it's quite likely
that your goals are not going to be optimally met by futzing with manual memory
management all day ([at least from the macro level, in the general
sense](https://danluu.com/sounds-easy/)).

But what if you _do_ need that control? Well then, _you need it_. When every
ounce of performance is necessary, or on embedded systems, or when writing code
for some bespoke or otherwise uncommon processor.

I'm going to choose one language, Zig, and dive deep into its hello world, but
it is important to note here that my point is not primarily about Zig, it's
about how all languages have to contend with an enormous amount of complexity
in order to do _anything_, even the simplest of tasks like a hello world
program. Complexity that is, for the most part, hidden from us in our day to
day.  So what in the hello world is _actually_ going on then?

> I'll be using the most current minor release version of Zig: 0.6.0.

Let's take a walk
---------------

[Zig](https://ziglang.org/)'s hello world looks like this, from the docs.:

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().outStream();
    try stdout.print("Hello, {}!\n", .{"world"});
}
```

If you are new to zig, a quick word on this syntax before I get into the gritty
details.

```zig
const std = @import("std");
```

`@import` is a [_compiler
builtin_](https://ziglang.org/documentation/0.6.0/#Builtin-Functions) function
that assigns the namespace of the file it is referencing to the `const`
variable on the left hand side.

```zig
pub fn main() !void {
  //...
}
```
Just like in C, `main` is a special function that marks the entry point to a
program after it has been compiled as an executable. Unlike in C, it accepts no
arguments (C's main function has a variety of vagaries that make it a bit
[unique](https://stackoverflow.com/a/4207223)) and command line input is
available through utility functions to allow easier cross platform use.

It is marked [`pub`](https://ziglang.org/documentation/0.6.0/#Keyword-pub) so
that it is accessible from outside of the immediate module ('module' here
referring to nothing more than the top level scope of the current namespace...
i.e., the file), this is a necessary step since, as the program's entry point,
`main` would _have_ to be accessible from outside the immediate scope.

`fn` is the function keyword.

`main()` is the name of the function (and where the argument list _would_ be)
and `!void` is the return type. Looking a little closer at that return type:

In C, the return type of a function is declared _before_ anything else. This
makes a certain amount of sense: it's congruent with how variables are
declared, after all, and scanning the file you can see clearly "calling this
will get you that."

In Zig, the return type comes after the function declaration but before the
function body. This also makes sense! It's the same in Rust and Go, and seems
to be generally a more modern approach. The reason is actually pretty simple:
doing it this way makes it possible to have a context-free grammar! C and C++
put the parser in a position where it [has to understand semantics to even just
_parse_ the source
code.](https://stackoverflow.com/questions/14589346/is-c-context-free-or-context-sensitive)

In Zig, `main` returns `void` (well, actually, it can return a variety of
things, and if it returns void (which is just a way of saying it doesn't return
anything at all)), it's actually returning
[`0`](https://github.com/jfo/zig/blob/7381aaf70e0cad92fc52b79f3aa2a0abb7c3ee04/lib/std/start.zig#L241-L244)
as a success code, but) there is a wrinkle!  `void` is preceded by an
exclamation mark. This means: "This function is supposed to return `void`, but
it _could_ fail and return an error." This is an [inferred error
set](https://ziglang.org/documentation/0.6.0/#Inferred-Error-Sets), and
whenever a function that _could_ fail is called, the compiler will enforce that
you handle that error at the call site. More on Zig's error handling some other
time, for now it is enough to understand what the `!` in front of the return
type declaration means. I want to move on to the body of the function, let's go
line by line.

```zig
const stdout = std.io.getStdOut().outStream();
```

So, we can see that this is a call into a standard library function (`std`)
that returns something that we assign to `const stdout`. Standard out (stdout) and
standard err (stderr) may be familiar concepts from the shell, but what does it
mean to be referring to `stdout` here in this program? What exactly _is_
`stdout`? Whatever it is, it's being returned by the call to `outStream()`,
which is a method called on the return value of `std.io.getStdOut()`, so we
first need to know what _that_ is.

To the source! In the Zig source tree, `std` lives in `lib/std/std.zig`, which
is a file that makes a wide variety of functionality available. It includes the line:

```zig
pub const io = @import("io.zig");
```

Which is referred to on the `std` variable as `std.io` (again, notice the `pub`
keyword, without which this declared constant would be inaccesible outside of
this immediate scope). Going deeper, into `lib/std/io.zig`...

```zig
pub fn getStdOut() File {
    return File{
        .handle = getStdOutHandle(),
        .capable_io_mode = .blocking,
        .intended_io_mode = default_mode,
    };
}
```

So, `stdout` is a _File_ struct. Let's look at that. It is imported at the top
of `io.zig` as

```zig
const File = std.fs.File;
```

and lives in the source, perhaps unsurprisingly, at `lib/std/fs/File.zig`. This
struct definition is quite long, so I'll focus on what we want to look at, the
`outStream()` method.


An aside: methods vs functions
-----

Zig doesn't _really_ have methods, but it's useful to talk about a special
class of functions _as_ methods, since the calling convention supports implicit
passing of `self` when called on a struct "instance" using dot syntax. Let me
show you what I mean.

```zig
const std = @import("std");

const Thing = struct {
    instanceVariable: u8,
    const classVariable = 41;

    fn staticMethod(y: u8) u8 {
        return classVariable + y;
    }

    fn instanceMethod(self: Thing) u8 {
        return self.instanceVariable;
    }
};

pub fn main() !void {
    std.debug.warn("{}\n", .{ Thing.staticMethod(1) }); // 42
    const thing = Thing{ .instanceVariable = 1 };
    std.debug.warn("{}\n", .{ thing.instanceMethod() }); // 1
}
```

So, despite the lack of explicit _classes_, these patterns are available
because of support for this calling convention. Treating a `struct` _like_ a
class definition, you can call a "static method" on the struct definition
_itself_. In the example above,

```zig
Thing.staticMethod(1);
```

Is equivalent to the

```ruby
Thing::staticMethod
```

syntax in Ruby. In fact, the equivalent example in Ruby looks startingly
similar to the Zig version:

```ruby
class Thing
  attr_accessor :instanceVariable
  @@classVariable = 41

  def initialize(instanceVariable)
    @instanceVariable = instanceVariable
  end

  def self.staticMethod(y)
    @@classVariable + y
  end

  def instanceMethod()
    @instanceVariable
  end
end

p Thing::staticMethod(1) # 42
thing = Thing.new(1)
p thing.instanceMethod # 1
```
There are of course notable differences here! Attempting to call a static
method on an _instance_ of a class in ruby

```ruby
p thing.staticMethod 2
```
will not get you very far

```
thing.rb:21:in `<main>': undefined method `staticMethod' for #<Thing:0x0000000002284d38 @instanceVariable=1> (NoMethodError)
```

Likewise, the other way:

```ruby
p Thing::instanceMethod(1)
```
```
thing.rb:18:in `<main>': undefined method `instanceMethod' for Thing:Class (NoMethodError)
```

Ruby is a full throated object oriented language, and so of course its
underlying class abstraction is more robust than this facsimile of it in Zig,
but the effect of that is that, well, there's really nothing special about a
zig "instance" vs "static" method, as they are simply functions defined on the
struct that _happen_ to be available through multiple calling conventions.

Take this again, with the same `Thing` struct definition from above:

```zig
const thing = Thing{ .instanceVariable = 1 };
std.debug.warn("{}\n", .{ thing.staticMethod() });
```
You will get a compiler error:

```
./thing.zig:19:31: error: expected type 'u8', found 'Thing'
    std.debug.warn("{}\n", .{ thing.staticMethod() });
```

But it's telling you that you passed a `Thing` to the method. This is the
important point: "instance methods" have special access to "instance variables"
because they have a reference to the struct they are being called on, that's
all.  That's all the magic there is here.

> Note also that there is nothing special about the word '_self_', it is
> just a conventional variable name.

For completeness, the other direction:

```zig
std.debug.warn("{}\n", .{ Thing.instanceMethod() });
```

You will get what you might expect, given the last example:

```
./thing.zig:17:51: error: expected 1 arguments, found 0
    std.debug.warn("{}\n", .{ Thing.instanceMethod() });
```

No implicit passing of `self` means an arity error on this call.

But, to underscore the fact that there is nothing magical happening here, you
can indeed do this:

```zig
const thing = Thing{ .instanceVariable = 1 };
std.debug.warn("{}\n", .{ thing.instanceMethod() });
std.debug.warn("{}\n", .{ Thing.instanceMethod(thing) });
```

Those two calls to`instanceMethod` are the same, but with differing calling
conventions (and so the first one passes `self` implicitly!)


The `outStream()` "method"
--------------

Back in `lib/std/fs/File.zig`, we see the definition of this "instance method"

```zig
pub fn outStream(file: File) OutStream {
  return .{ .context = file };
}
```

This is returning an `OutStream` struct that is initialized with `self` of the
`File` it was called on (here referred to as `file`). Zig supports [anonymous
struct
literals](https://ziglang.org/documentation/0.6.0/#Anonymous-Struct-Literals)
and in this case is able to infer the type based on the return value of the
function. Note too the odd syntax of starting an anonymous struct literal with
`.`, which is to syntactically distinguish it from a block.

So, further down again, what is an `OutStream`? Its definition is just above:

```zig
pub const OutStream = io.OutStream(File, WriteError, write);
```

Hmm, this is interesting... is this function call returning a... type
definition? That is then assigned to `OutStream` and used as a return value for
`pub fn outStream`?

That's exactly what it's doing! In `lib/std/io/outStream.zig`:

```zig
pub fn OutStream(
    comptime Context: type,
    comptime WriteError: type,
    comptime writeFn: fn (context: Context, bytes: []const u8) WriteError!usize,
) type {
    return struct {
        context: Context,
        //...
    }
}
```

This is Zig's way of supporting generics! Given some compile time known values,
you can create a struct definition _on the fly, at compile time_. Here is a
more detailed post about that capability: [What is Zig's
Comptime?](https://kristoff.it/blog/what-is-zig-comptime/).

For now, take careful note that `write` is being passed to `io.OutStream` as
the `writeFn` argument, which will eventually be what is called to print to
standard out.

Alright, phew, so that's

```zig
const stdout = std.io.getStdOut().outStream();
```

We've ended up with an `OutStream` struct with its `context` field initialized
to the `File` struct returned by `std.io.getStdOut()`.

Now for the money business.

```zig
try stdout.print("Hello, {}!\n", .{"world"});
```

The definition of this "instance method" lives in `lib/std/io/outStream.zig`.

```zig
pub fn print(self: Self, comptime format: []const u8, args: var) Error!void {
    return std.fmt.format(self, format, args);
}
```
This dispatces `self` to `std.fmt.format` along with two more arguments. Let's
look at that function

```zig
pub fn format(
    out_stream: var,
    comptime fmt: []const u8,
    args: var,
) !void {
  //...
}
```

Ok we're getting closer: `out_stream` is in our case, the `File` from way back
at the beginning.

> [Andy](https://andrewkelley.me) said: "Almost - it's the file.outStream()
> return value. Which is just the "Context" with the write function as part of
> the type. The way streams work in zig right now is with "duck typing". It
> optimizes well, the API is mostly good, but it can produce bloated code, and
> in some cases the API is annoyingly too generic. Sometimes it would be nice
> to accept a non-"var" type as a stream parameter."

The other two arguments are being passed in at the top level
call site, a string constant and an [anonymous list
literal](https://ziglang.org/documentation/0.6.0/#Anonymous-List-Literals)
(whose behavior is unsurprisingly similar to the aformentioned anonymous struct
literal) of positional arguments meant to be interpolated into the format
string at points marked by `{}`. You can pass in [formatting
options](https://ziglang.org/documentation/0.6.0/std/#std;fmt.format) much like
c's `printf`.

`format` is a long function, there is a lot of bookeeping going on, but the
meat of it are its calls to `out_stream.writeAll`. Jumping back to that
definition:

```zig
pub fn writeAll(self: File, bytes: []const u8) WriteError!void {
    var index: usize = 0;
    while (index < bytes.len) {
        index += try self.write(bytes[index..]);
    }
}
```

We can see that it calls into `self.write`, which looks like:

```zig
pub fn write(self: File, bytes: []const u8) WriteError!usize {
    if (is_windows) {
        return windows.WriteFile(self.handle, bytes, null, self.intended_io_mode);
    } else if (self.capable_io_mode != self.intended_io_mode) {
        return std.event.Loop.instance.?.write(self.handle, bytes);
    } else {
        return os.write(self.handle, bytes);
    }
}
```

And now, finally, we're down to the `system` in `systems programming` This
method operates differently depending on the system it's being used on! At the
top of this file `lib/std/fs/file.zig`,

```zig
const is_windows = std.Target.current.os.tag == .windows;
```

I am not on windows, and I will for now ignore the second branch so I don't
have to get into `async`, so I end up here:

```zig
return os.write(self.handle, bytes);
```

I am calling into an os specific library function that accepts a place to write
bytes and bytes to write (by this point formatted with those interpolated
values from the call site). Here's where it gets good.

`os.write` calls into `system.write` which is defined _per architecture_

```zig
pub const system = if (@hasDecl(root, "os") and root.os != @This())
    root.os.system
else if (builtin.link_libc)
    std.c
else switch (builtin.os.tag) {
    .macosx, .ios, .watchos, .tvos => darwin,
    .freebsd => freebsd,
    .linux => linux,
    .netbsd => netbsd,
    .dragonfly => dragonfly,
    .wasi => wasi,
    .windows => windows,
    else => struct {},
};
```

For me, that ends up being `linux`, defined here:

```zig
pub const linux = @import("os/linux.zig");
```
So in my case, `system.write` ends up being:

```zig
pub fn write(fd: i32, buf: [*]const u8, count: usize) usize {
    return syscall3(.write, @bitCast(usize, @as(isize, fd)), @ptrToInt(buf), count);
}
```

where `syscall3` is imported directly into the namespace according to
architecture:

```zig
pub usingnamespace switch (builtin.arch) {
    .i386 => @import("linux/i386.zig"),
    .x86_64 => @import("linux/x86_64.zig"),
    .aarch64 => @import("linux/arm64.zig"),
    .arm => @import("linux/arm-eabi.zig"),
    .riscv64 => @import("linux/riscv64.zig"),
    .mips, .mipsel => @import("linux/mips.zig"),
    else => struct {},
};
```

> here, the "3" in _syscall3_ refers to the number of "arguments" required for
> the syscall being invoked.

for me, that's `x86_64`, and it looks like this:

```zig
pub fn syscall3(number: SYS, arg1: usize, arg2: usize, arg3: usize) usize {
    return asm volatile ("syscall"
        : [ret] "={rax}" (-> usize)
        : [number] "{rax}" (@enumToInt(number)),
          [arg1] "{rdi}" (arg1),
          [arg2] "{rsi}" (arg2),
          [arg3] "{rdx}" (arg3)
        : "rcx", "r11", "memory"
    );
}
```

This is _close to_ but not _exactly_ what the compiler will _actually emit_ for
this call. Amazingly, even all the way down here, where we're seeing register
names invoked directly, there is still a layer of abstraction within which the
compiler has some wiggle room.

Not so _simple_ a program now, is it? Remember, these syscalls differ for each
architecture, the compiler produces machine code based on what you're
targeting, so this is just one of many possible paths. I think that it is very
easy to forget how complicated this can quickly become when you poke at it!

Bottoms up
-----------

Let's come at this from a slightly different angle now. We know that the
Zig compiler's job, just like any compiler, is to take _source code_ and turn it
into something else. Zig is highly portable; using llvm as a backend means
it can target basically anything that llvm targets, with the caveat that not
all library functions will have the same amount of support on all platforms
(see the [support
table](https://ziglang.org/download/0.6.0/release-notes.html#Tier-System)) for
more detail on that).

So, there are many possible targets, and so there are many possible "something
else"s for the source to be turned into. But let's look at the most obvious
case: building an executable that targets my current running system.

The transformation pipeline inside the compiler goes from `source` ->
`intermediate representation(s)` -> `target`, where `intermediate
representations` could be many things and include many steps. Zig has its own
IR, as a matter of fact, on which it runs its own static analysis processes
before transforming it to LLVM IR and passing it along where is could be
processed through [many possible optimazation and compilation/assemblage
steps](http://llvm.org/docs/Passes.html) (LLVM calls these "passes").
`target`, for me, is x86_64 machine code, but the last stop _before_ that,
conceptually, is assembly.

Because clang is a full compiler toolchain built on llvm, and Zig can be used
as a [drop in replacement for
clang](https://andrewkelley.me/post/zig-cc-powerful-drop-in-replacement-gcc-clang.html),
we should be able to use `zig cc` to compile assembly code directly into machine code.
This step is actually called _assembling_, not compiling, and is done by an
"assembler" but tbh this is a [distinction without much of a
difference](http://composition.al/blog/2017/07/30/what-do-people-mean-when-they-say-transpiler/)

What is the advantage of using `zig cc`? Primarily that you are able to
reliably use the same toolchain and version of llvm that the version of zig you
are using relies on. No futzing around with system libraries and linkages, it's
all just ready to work.

So! I make an empty file:

```
$ touch hello.s
```

`clang` is smart enough to detect a filetype by its extension, so, so is `zig cc`.

```
$ zig cc hello.s
```

```
zig: warning: argument unused during compilation: '-nostdinc' [-Wunused-command-line-argument]
zig: warning: argument unused during compilation: '-fno-spell-checking' [-Wunused-command-line-argument]
zig: warning: argument unused during compilation: '-fno-omit-frame-pointer' [-Wunused-command-line-argument]
zig: warning: argument unused during compilation: '-D _DEBUG' [-Wunused-command-line-argument]
zig: warning: argument unused during compilation: '-fstack-protector-strong' [-Wunused-command-line-argument]
zig: warning: argument unused during compilation: '--param ssp-buffer-size=4' [-Wunused-command-line-argument]
zig: warning: argument unused during compilation: '-isystem /usr/local/include' [-Wunused-command-line-argument]
zig: warning: argument unused during compilation: '-isystem /usr/include/x86_64-linux-gnu' [-Wunused-command-line-argument]
zig: warning: argument unused during compilation: '-isystem /usr/include' [-Wunused-command-line-argument]
lld: error: undefined symbol: main
>>> referenced by start.S:104 (/home/jfo/code/zig/build/lib/zig/libc/glibc/sysdeps/x86_64/start.S:104)
>>>               /home/jfo/.cache/zig/stage1/o/ujWleITFBRHwV19Tq0gsSK_F_gRDc7-jgOCip86Un1bhdmx0pIXLGQRxtjMtuntC/Scrt1.o:(_start)

```

Most of this is just telling us that the flags zig is passing to clang by
default weren't used for anything, which isn't much of a surprise since there
was nothing to compile! (Strictly speaking, this is a bug in the zig compiler,
but for our purposes it has no effect) There is a _real_ error here, too.

```
lld: error: undefined symbol: main
```

`lld` is the [_linker_](https://lld.llvm.org/) bundled with llvm bundled with
clang, and so bundled with zig, and it is complaining that this program (which
is empty) that we're trying to turn into an executable doesn't have an entry
point. How would you run it? Where would you start? A reasonable complaint.

We can instead build an "object file" that isn't intended to be executable by
passing the `-c` flag.

> These options are the _same_ options as clang, as all of these arguments are
> simply being forwarded to clang along with the compiler flags set by zig as defaults.

```
$ zig cc -c hello.s
```

This throws all the same warnings as before, but it succeeds, and produces
`hello.o`, an object file.

Running `file` on this output

```
$ file hello.o
```

Will tell us what we have got.

```
hello.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped
```

This is essentially a bundle of machine code that would be suitable for linking
into other programs during their [own linking
phase](https://stackoverflow.com/questions/24655839/what-is-the-difference-between-executable-and-relocatable-in-elf-format),
if there was actually any code in there at all to be used. As it stands,
there's just the [ELF header](https://lwn.net/Articles/631631/) to identify the
file type and what I assume to be a bit of metadata and some padding.

But we wanted to actually make an executable, so we need an entry point! What
does an entry point look like?

In x86 assembly, the default entry point looks like:

```
_start:
```

I add that to the file, and try again:

```bash
$ zig cc hello.s
```

and

```
lld: error: undefined symbol: main
>>> referenced by start.S:104 (/home/jfo/code/zig/build/lib/zig/libc/glibc/sysdeps/x86_64/start.S:104)
>>>               /home/jfo/.cache/zig/stage1/o/ujWleITFBRHwV19Tq0gsSK_F_gRDc7-jgOCip86Un1bhdmx0pIXLGQRxtjMtuntC/Scrt1.o:(_start)
```

(I have left out the assembler warnings from above).

What? If the default entry point is `_start`, why is it asking for `main`, then?

Looking at the error, it _is_ trying to load `_start`, just not _our_ start.
When you compile a regular zig or c program with a `main` function, your
program doesn't _actually_ start at main, it _also_ starts at `_start`, which
is responsible for doing memory setup and generally getting everything tidy for
you before your `main` function runs. Remember, here we're just using `zig cc`
as a pass through for clang, and so the _real_ definition of `_start` resides
in libc, and looks like
[this](https://github.com/jfo/zig/blob/master/lib/libc/glibc/sysdeps/x86_64/start.S).

There are two ways I can solve this now. First, I can just make an assembly
file with `main:` instead... let's try that.

```asm
main:
```
```bash
$ zig cc hello.s
```

```bash
lld: error: undefined symbol: main
>>> referenced by start.S:104 (/home/jfo/code/zig/build/lib/zig/libc/glibc/sysdeps/x86_64/start.S:104)
>>>               /home/jfo/.cache/zig/stage1/o/ujWleITFBRHwV19Tq0gsSK_F_gRDc7-jgOCip86Un1bhdmx0pIXLGQRxtjMtuntC/Scrt1.o:(_start)
```

Ah, remember `pub fn main()`? In addition to being defined, this symbol also
needs to be available to the linker.  In assembly, that means putting it as a
`.globl` declaration.

```asm
.globl main
main:
```

Without that, the assembler is free to discard or mangle the label since it
assumes it's not needed for any other steps.

```bash
$ zig cc hello.s
```

This assembles! When I run the resulting executable, I get:

```
Trace/breakpoint trap (core dumped)
```

This isn't surprising, I've written a program that does nothing. I'm not
particularly interested in this error right now, the important thing is that I
got this to assemble.

I'm also not interested in using the libc startup code and `_start` call; I
want to do everything myself.

I will try to definie my own `.globl _start`:

```asm
.globl _start
_start:
```

```bash
$ zig cc hello.s
```

```
lld: error: duplicate symbol: _start
>>> defined at start.S:63 (/home/jfo/code/zig/build/lib/zig/libc/glibc/sysdeps/x86_64/start.S:63)
>>>            /home/jfo/.cache/zig/stage1/o/ujWleITFBRHwV19Tq0gsSK_F_gRDc7-jgOCip86Un1bhdmx0pIXLGQRxtjMtuntC/Scrt1.o:(_start)
>>> defined at zig-cache/o/UbeRHF13NXWYrl3f0cuxy3OffVcjvaAbWr5e2svkLcYXPpPG03d4JplnyJU7tFJ7/empty.o:(.text+0x0)
```

Given what we've seen so far, this makes complete sense. `_start` has already
been defined in the libc code, so simply putting it here results in this error,
because of course it does.

The linker takes an option to simply not use anything in the standard library,
which is exactly what I want.

```bash
$ zig cc -nostdlib hello.s
```
```
./a.out
```

```
Segmentation fault (core dumped)
```

Hello world!

----

Alright, now I've sussed out the precise incantations to go directly from an
x86 assembly file (`.s`) to an executable. What is the _smallest assembly
program I can write?_

That would be a program that simply exits.

```asm
.intel_syntax noprefix
.globl _start

_start:
  mov     rax, 60
  syscall
```

> I am using inter syntax here, the first line tells the assembler that.  An
> interesting note here, is that llvm [doesn't need you to explicitly say `noprefix`](https://github.com/llvm-mirror/llvm/blob/2c4ca6832fa6b306ee6a7010bfb80a3f2596f824/lib/Target/X86/AsmParser/X86AsmParser.cpp#L3563-L3584)
> but `gcc` _does_, so it makes sense to always use it.

What's happening here? I'm just putting a static value: `60`, into the `rax`
register, and then making a `syscall`. The syscall looks at the `rax` register
and does what the value inside of it corresponds to, which is `sys_exit`, so
the program exits, that's it.

When I compile and run this program, nothing happens, but
[`strace`](https://jvns.ca/blog/2015/04/14/strace-zine/) tells me that it's
doing exactly what I expected:

```
zig cc -nostdlib empty.s && strace ./a.out
```

```
execve("./a.out", ["./a.out"], 0x7ffd47b269c0 /* 104 vars */) = 0
exit(0)                                 = ?
+++ exited with 0 +++
```

Futhermore, the `sys_exit` syscall looks in the `rdi` register to get the
_value_ it returns to the calling process. I can put whatever I want in there
before executing the syscall.

```asm
.intel_syntax noprefix
.globl _start

_start:
  mov     rdi, 0xface
  mov     rax, 60
  syscall
```


```
execve("./a.out", ["./a.out"], 0x7fffa91a9be0 /* 104 vars */) = 0
exit(64206)                             = ?
+++ exited with 206 +++
```

You'll notice, that even though I loaded a 32 bit value into the register, and
`r` prefixed registers are 64 bits wide (todo citation), it only looked at the
bottom 8 bits. This leads me to believe that error codes must be between 1 and
255, is this correct? Yes it is. todo citation

Running this without strace, nothing happens, which surprised me, actually. I
thought that `0` was a success code and anything else was an error code. This
is [conventionally the
case](https://github.com/jfo/zig/blob/7381aaf70e0cad92fc52b79f3aa2a0abb7c3ee04/lib/libc/include/generic-glibc/stdlib.h#L91-L92)!
But it's up to the _caller_ to interpret that code and respond to it. For my
little program, there is no error handling that reports back to the user what's
going on, it just dutifully exits with the value I gave it and that's that.

Though the origins of the hello world are well known (it was the very [first
example](https://en.wikipedia.org/wiki/%22Hello,_World!%22_program) in the
hugely influential K&R C book), the definition of a hello world has grown over
the years. Though the canonical example is still "output the literal text
'Hello World!'", lots of paradigms and systems have their own version of it- my
favorite is the arduino [blink](https://www.arduino.cc/en/Tutorial/Blink); it's
the simplest thing you can do with it that proves it's working as intended.

By that measure, the above code is _already_ an assembly hello world in that
we've compiled it and proved it works! But it's a very small step to get to a
_real_ hello world, too.

Here it is!

```asm
.intel_syntax noprefix
.globl  _start

_start:
  mov     rax, 0x1
  mov     rdi, 0x1
  lea     rsi, msg
  mov     rdx, 14
  syscall
  xor     rdi, rdi
  mov     rax, 60
  syscall
msg:
  .ascii  "Hello, world!\n"
```

You can see that there's not really that much more here. A little more preamble
at the top, and then _two_ syscalls instead of one, followed by a little data
section that actually holds the text we're printing to the screen. Let's go
through this, line by line.

```asm
.intel_syntax noprefix
```

Trying to compile in gcc gives me errors:

```
hello.s: Assembler messages:
hello.s:4: Error: ambiguous operand size for `mov'
hello.s:5: Error: ambiguous operand size for `mov'
hello.s:6: Error: too many memory references for `lea'
hello.s:7: Error: ambiguous operand size for `mov'
hello.s:9: Error: ambiguous operand size for `mov'
hello.s:10: Error: too many memory references for `xor'
```

but it works just fine with clang.

```asm
.globl  _start
```

We know this one: marking the `_start` symbol available to the linker.

Next, on the the body of the program:

```asm
_start:
  mov     rax, 0x1
  mov     rdi, 0x1
  lea     rsi, msg
  mov     rdx, 14
  syscall
```

Looking at [this chart of linux
syscalls](https://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/),
I can see that what is going to be in `rax` when I reach `syscall` is `1`,
which corresponds to the `SYSWRITE` system call. Its "arguments" live in the
registers `rdi`, `rsi`, and `rdx`:


todo: table markdown
```
%rax	System call	%rdi	%rsi	%rdx
1	sys_write	unsigned int fd	const char *buf	size_t count
```

Where `fd` (file descriptor) is for the _output stream_, `buf` is a _pointer to
the buffer from which we want to write_ and `count` is the _number of bytes we
want to write_.

What are we putting into those registers, then?

In `rax`, we insert the syscall number for sys_write: `1`

`rdi`: The file descriptor for `stdout`, [defined by
POSIX](https://en.wikipedia.org/wiki/Standard_streams#Standard_output_(stdout))
to be `1`

`rsi`: a pointer to the buffer we write from. Here, we see the [`lea`
instruction](https://stackoverflow.com/a/57796189/2727670) for "load effective
address", and the right hand value is a label `msg` that is defined at the bottom of the file:

```
msg:
  .ascii  "Hello, world!\n"
```

Finally, in `rdx` we put in `14`, the length of the buffer. It is not runtime
known, since there is no runtime!

With all of these loaded into the appropriate registers,

```
syscall
```

Executes `sys_write` and writes the contents of the memory buffer to stdout.

The remaining three lines:

```asm
xor     rdi, rdi
mov     rax, 60
syscall
```

Might be familiar; we're loading something into `rdi` and calling `sys_exit`
(60), just like the first example. In this case though, we're `xor`ing `rdi`
with itself, which is the same thing as setting it to `0`: the success code.

And that's it!

coda:
-----

My intention now was to find _exactly_ where the zig compiler emits these
_exact_ instructions. And it _does_ do that, sort of... I mean, it _has_ to,
because these are the instructions to do this operation on x86. You may have
noticed, too, that the pertinent registers figure prominently in the `syscall3`
inline asm from above... `rax`, `rdi`, `rdx`, and `rsi` as well:

```
// ...
        : [number] "{rax}" (@enumToInt(number)),
          [arg1] "{rdi}" (arg1),
          [arg2] "{rsi}" (arg2),
          [arg3] "{rdx}" (arg3)
// ...
```

But as alluded to earlier, even here at this lowest of levels (I mean, this is
_almost_ machine code, right?) there is flexibility for the compiler to be
creative. And I suppose the specifics of the compiler's output from this command:

```
$ zig build-exe hello.zig -femit-asm --strip --single-threaded --release-small
```

are in the end a bit extraneous to my ultimate point, which is that _hello
world is complicated_.

It is, canonically, the "simplest" program you can write, and it is built on top
of _heaps_ of abstract complexity that for the most part, none of us ever
really think about. Do you remember the first time you heard some systems
engineer refer to C as a high level langauge? Did it sound weird? Do you
remember the first time you realized that it actually _is_ a high level
language?
