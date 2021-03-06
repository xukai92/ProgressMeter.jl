# ProgressMeter.jl

[![Build Status](https://travis-ci.org/timholy/ProgressMeter.jl.svg?branch=master)](https://travis-ci.org/timholy/ProgressMeter.jl)

Progress meter for long-running operations in Julia

## Installation

Within julia, execute
```julia
Pkg.add("ProgressMeter")
```

## Usage

### Progress meters for tasks with a pre-determined number of steps

This works for functions that process things in loops or with map/pmap:

```julia
using ProgressMeter

@showprogress 1 "Computing..." for i in 1:50
    sleep(0.1)
end

@showprogress pmap(1:10) do x
    sleep(0.1)
    x^2
end
```

The first incantation will use a minimum update interval of 1 second, and show the ETA and final duration.  If your computation runs so quickly that it never needs to show progress, no extraneous output will be displayed.

The `@showprogress` macro wraps a `for` loop, comprehension, `@distributed` for loop, or map/pmap as long as the object being iterated over implements the `length` method and will handle `continue` correctly.

```julia
using Distributed
using ProgressMeter

@showprogress @distributed for i in 1:10
    sleep(0.1)
end

result = @showprogress 1 "Computing..." @distributed (+) for i in 1:10
    sleep(0.1)
    i^2
end
```

In the case of a `@distributed` for loop without a reducer, an `@sync` is implied.

You can also control progress updates and reports manually:

```julia
function my_long_running_function(filenames::Array)
    n = length(filenames)
    p = Progress(n, 1)   # minimum update interval: 1 second
    for f in filenames
        # Here's where you do all the hard, slow work
        next!(p)
    end
end
```

For tasks such as reading file data where the progress increment varies between iterations, you can use `update!`:

```julia
using ProgressMeter

function readFileLines(fileName::String)
    file = open(fileName,"r")

    seekend(file)
    fileSize = position(file)

    seekstart(file)
    p = Progress(fileSize, 1)   # minimum update interval: 1 second
    while !eof(file)
        line = readline(file)
        # Here's where you do all the hard, slow work

        update!(p, position(file))
    end
end
```

The core methods `Progress()`, `ProgressThresh()`, `ProgressUnknown()`, and their updaters
are also thread-safe, so can be used with `Threads.@threads`, `Threads.@spawn` etc.:

```julia
using ProgressMeter
p = Progress(10)
Threads.@threads for i in 1:10
    sleep(2*rand())
    next!(p)
end
```

```julia
using ProgressMeter
n = 10
p = Progress(n)
tasks = Vector{Task}(undef, n)
for i in 1:n
    tasks[i] = Threads.@spawn begin
        sleep(2*rand())
        next!(p)
    end
end
wait.(tasks)
```

### Progress bar style

Optionally, a description string can be specified which will be prepended to the output, and a progress meter `M` characters long can be shown.  E.g.

```julia
p = Progress(n, 1, "Computing initial pass...", 50)
```

will yield

```
Computing initial pass...53%|███████████████████████████                       |  ETA: 0:09:02
```

in a manner similar to [python-progressbar](https://code.google.com/p/python-progressbar/).

Also, other properties can be modified through keywords. The glyphs used in the bar may be specified by passing a `BarGlyphs` object as the keyword argument `barglyphs`. The `BarGlyphs` constructor can either take 5 characters as arguments or a single 5 character string. E.g.

```julia
p = Progress(n, dt=0.5, barglyphs=BarGlyphs("[=> ]"), barlen=50, color=:yellow)
```

will yield

```
Progress: 53%[==========================>                       ]  ETA: 0:09:02
```

It is possible to give a vector of characters that acts like a transition between the empty character
and the fully filled character. For example, definining the progress bar as:

```julia
p = Progress(n, dt=0.5,
             barglyphs=BarGlyphs('|','█', ['▁' ,'▂' ,'▃' ,'▄' ,'▅' ,'▆', '▇'],' ','|',),
             barlen=10)
```

might show the progress bar as:

```
Progress:  34%|███▃      |  ETA: 0:00:02
```

where the last bar is not yet fully filled.


### Progress meters for tasks with a target threshold

Some tasks only terminate when some criterion is satisfied, for
example to achieve convergence within a specified tolerance.  In such
circumstances, you can use the `ProgressThresh` type:

```julia
prog = ProgressThresh(1e-5, "Minimizing:")
for val in exp10.(range(2, stop=-6, length=20))
    ProgressMeter.update!(prog, val)
    sleep(0.1)
end
```

### Progress meters for tasks with an unknown number of steps

Some tasks only terminate when some non-deterministic criterion is satisfied. In such
circumstances, you can use the `ProgressUnknown` type:

```julia
prog = ProgressUnknown("Titles read:")
for val in ["a" , "b", "c", "d"]
    ProgressMeter.next!(prog)
    if val == "c"
        ProgressMeter.finish!(prog)
        break
    end
    sleep(0.1)
end
```
This will display the number of calls to `next!` until `finish!` is called.

If your counter does not monotonically increases, you can also set the counter by hand.
```julia
prog = ProgressUnknown("Total length of characters read:")
total_length_characters = 0
for val in ["aaa" , "bb", "c", "d"]
    global total_length_characters += length(val)
    ProgressMeter.update!(prog, total_length_characters)
    if val == "c"
        ProgressMeter.finish!(prog)
        break
    end
    sleep(0.5)
end
```

### Printing additional information

You can also print and update information related to the computation by using
the `showvalues` keyword. The following example displays the iteration counter
and the value of a dummy variable `x` below the progress meter:

```julia
x,n = 1,10
p = Progress(n)
for iter = 1:10
    x *= 2
    sleep(0.5)
    ProgressMeter.next!(p; showvalues = [(:iter,iter), (:x,x)])
end
```

### ProgressMeter in Jupyter notebooks

Since Jupyter notebooks don't allow to overwrite only parts of the output of cell, the progress bars are printed repeatedly to the output.
Jupyter notebooks allow to clear the output of a cell, but this will remove **all** output
of the current cell, i.e., to properly update a progress bar we need to wipe all output of cell. This behavior is enabled by default but you can
disable it by calling `ProgressMeter.ijulia_behavior(:append)`. 
You can enable it again by calling `ProgressMeter.ijulia_behavior(:clear)`, which will also disable the warning message.

### Tips for parallel programming

For remote parallelization, when multiple processes or tasks are being used for a computation, the workers should communicate back to a single task for displaying the progress bar. This can be accomplished with a `RemoteChannel`:

```julia
using ProgressMeter
using Distributed

p = Progress(10)
channel = RemoteChannel(()->Channel{Bool}(10), 1)

@sync begin
    # this task prints the progress bar
    @async while take!(channel)
        next!(p)
    end

    # this task does the computation
    @async begin
        @distributed (+) for i in 1:10
            sleep(0.1)
            put!(channel, true)
            i^2
        end
        put!(channel, false) # this tells the printing task to finish
    end
end
```

### `progress_map`

More control over the progress bar in a map function can be achieved with the `progress_map` and `progress_pmap` functions. The keyword argument `progress` can be used to supply a custom progress meter.

```julia
p = Progress(10, barglyphs=BarGlyphs("[=> ]"))
progress_map(1:10, progress=p) do x
    sleep(0.1)
    x^2
end
```

## Credits

Thanks to Alan Bahm, Andrew Burroughs, and Jim Garrison for major enhancements to this package.
