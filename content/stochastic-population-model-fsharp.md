+++
title = "Stochastic population model in F#"
date = 2020-11-08
[taxonomies]
tags = ["F#"]
+++

In [three](/programming-horizons/) [previous](/exploring-scheme-implementations/) [posts](/programming-horizons-revisited/), I wrote about different programming languages that I have considered learning. I mentioned about 15 different languages in those posts. [F#](https://fsharp.org/) was not on the list. Because my background is in R, I thought I was better off sticking to learning dynamically typed languages at this point. Moreover, I hold a longstanding bias against Microsoft and Windows and that bias was easy to transfer to F#. 

<!-- more -->

A few months ago, I started a new job. My work primarily involves R, but there is a software developer at the new company who primarily uses C#. Because I'm interested in eventually shifting my role to involve more software development, I thought it might be worthwhile to invest some time in learning C#. But, after a little more reading about C# and .NET, I found myself more drawn to F# as a functional-first language with strengths in data science (relative to C#).

Around the time that I started following the F# community, I saw a call for applications to the [F# Mentorship Program](https://fsharp.org/mentorship/). I decided to apply and was assigned a mentor. I've thoroughly enjoyed the mentorship program and, although I still know very little F#, I can see my appreciation of F# growing to the level of R and Scheme.   

I had [previously written](/stochastic-population-model-r-racket/) about implementing a stochastic population model in Racket based on an R example found in [this blog post](https://www.seascapemodels.org/rstats/2017/02/26/speeding-up-sims.html). As a first exercise in F#, I've translated the stochastic population model into a few F# versions.

I'm working with F# in Visual Studio Code with [Ionide](https://ionide.io/) on Linux (Ubuntu), but I think that the code shared in this post will work on other platforms. We can initialize a new console app with the following line.

```
dotnet new console -lang "F#" -o StochasticLogistic
```

That line creates a folder `StochasticLogistic` containing the files `Program.fs` and `StochasticLogistic.fsproj` and  a folder `obj`. The `Program.fs` file for the code described in this blog post is available in [this gist](https://gist.github.com/hinkelman/d089e62f5bdbfe4b8520f1af20bc41e3). 

We will use the [`MathNet.Numerics`](https://www.nuget.org/packages/MathNet.Numerics/) package for drawing random numbers. One way to install the package is with `dotnet`.

```
dotnet add package MathNet.Numerics --version 4.12.0
```

The first lines of `Program.fs` open the `Distributions` namespace of `MathNet.Numerics` and set the values of the parameters used in the model. 

```
open MathNet.Numerics.Distributions

[<EntryPoint>]
let main argv =
    let yinit = 1.0            // initial population size
    let r = 1.4                // maximum population growth rate
    let k = 20.0               // carrying capacity
    let thetasd = 0.1          // standard deviation for adding noise to the population
    let t = 4                  // number of years of growth to simulate
```

The first version of the function, `logmodFor`, follows the R example by using a `for` loop to fill an array. In F#, we draw random numbers by initializing a distribution and then sampling from the distribution. The `theta` variable is the stochastic part of the model. In this version, we initialize a 1D array with random draws from a normal distribution (following the R example). Alternatively, we could have drawn one value every time through the `for` loop. `Array.init` illustrates the syntax for anonymous functions in F#, e.g., (fun x -> x + 1). F# infers types from context. In the case of `Array.zeroCreate`, though, we need to specify the type (`float`) of the zeros in the array. We initialize the 0th element of the output array `ys` to the initial population value `y` and fill the rest of the array each time through the loop.

```
    let logmodFor t y r k thetasd = 
        let normalDist = Normal(0.0, thetasd)
        let theta = Array.init (t - 1) (fun _ -> normalDist.Sample())
        let ys: float array = Array.zeroCreate t
        Array.set ys 0 y
        for i in 1 .. (t - 1) do
            Array.set ys i (ys.[i-1] * (r - r * (ys.[i-1] / k)) * exp(theta.[i-1]))
        ys
```

The next version of the function, `logmodRecIf`, uses recursion and is written in a style similar to how I tend to write recursive functions in Scheme. I moved the main calculation into a separate function `calc` to improve readability. Note the use of `rec` to indicate the recursive function `loop`. `::` is the cons operator. `calc acc.Head` could also have been written as `calc (List.head acc)`, but dot notation is one of the [recommended](https://docs.microsoft.com/en-us/dotnet/fsharp/style-guide/conventions#object-programming) object-oriented features to include in your F# code. 

```
    let logmodRecIf t y r k thetasd =
        let normalDist = Normal(0.0, thetasd)
        let calc y = y * (r - r * (y / k)) * exp(normalDist.Sample())
        let rec loop acc t = 
            if t = 1 then List.rev acc
            else loop ((calc acc.Head) :: acc) (t - 1)
        loop [y] t
```

The next version, logmodRecMatch, is only slightly different than the last. In this version, we are using pattern matching instead of `if then else`. Even in this simple example, which doesn't require the power of `match`, I find the pattern matching version more readable. With `match`, we are first checking if the value of `t` is `1` and, if not, recursing through the rest of the list. 

```
    let logmodRecMatch t y r k thetasd = 
        let normalDist = Normal(0.0, thetasd)
        let calc y = y * (r - r * (y / k)) * exp(normalDist.Sample())
        let rec loop acc t = 
            match t with 
            | 1 -> List.rev acc
            | _ -> loop ((calc acc.Head) :: acc) (t - 1)
        loop [y] t
```

The next version, `logmodUnfold`, was provided by my F# mentor. I don't fully understand it, yet. For example, I'm not sure (and don't remember what I was told) if using `Some` was a convenient way to make the function type check or if it represents best practice for this use case. This approach uses sequences, which are lazy. F# also provides `List.unfold` and `List.take`, but they are not drop-in replacements because lists are not lazy and `calc` as written here never terminates. 

```
    let logmodUnfold t y r k thetasd = 
        let normalDist = Normal(0.0, thetasd)
        let calc y = 
            let result = y * (r - r * (y / k)) * exp(normalDist.Sample())
            Some (y, result)
        Seq.unfold calc y |> Seq.take t
```

Now that we have our functions we need call them in our program and display the output in the console. Because the model has a stochastic component, it is not obvious if all four versions of the function are performing the same calculation. I wanted to be able to run deterministic or stochastic versions of the functions when calling the program from the command line. I also decided to provide the number of years `t` as a command line argument to the program.

The arguments provided to the program are collected in a string list `argv`. We will pass `t` as the first argument and convert the string to an integer with `int`. If the `t` argument can't be converted to an integer, then the program will produce a runtime error.

```
[<EntryPoint>]
let main argv =
    let yinit = 1.0            // initial population size
    let r = 1.4                // maximum population growth rate
    let k = 20.0               // carrying capacity
    let thetasd = 0.1          // standard deviation for adding noise to the population
    let t = int argv.[0]       // number of years of growth to simulate
    let seedType = argv.[1]    // "fixed" or "random" for setting deterministic behavior
```
 
 The second `argv` allows for specifying fixed or random behavior. To use `seedType`, I added two new lines to each function and changed the `Normal` function to use the `RandomSource` parameter. 

```
        let random = System.Random()
        let seed = if seedType = "fixed" then 2005 else random.Next()
        let normalDist = Normal(0.0, thetasd, RandomSource = System.Random(seed))
```

The following block of code runs our four function versions and pipes `|>` the output into a `printfn`. Note, the `|>` operator pipes the previous output into the last argument of the next function. `"%A"` is used to print any F# object. 

```
    printfn "for loop"
    logmodFor t yinit r k thetasd seedType |> printfn "%A"
    printfn "---"
    printfn "recursion (if)"
    logmodRecIf t yinit r k thetasd seedType |> printfn "%A"
    printfn "---"
    printfn "recursion (match)"
    logmodRecMatch t yinit r k thetasd seedType |> printfn "%A"
    printfn "---"
    printfn "unfold"
    logmodUnfold t yinit r k thetasd seedType |> printfn "%A"
```

We can run our program with `dotnet run` and the required arguments. 

```
$ dotnet run 4 fixed
for loop
[|1.0; 1.305926731; 1.454837341; 1.798817442|]
---
recursion (if)
[1.0; 1.305926731; 1.454837341; 1.798817442]
---
recursion (match)
[1.0; 1.305926731; 1.454837341; 1.798817442]
---
unfold
seq [1.0; 1.305926731; 1.454837341; 1.798817442]
```

`dotnet run` compiles the program before running it. If you want to re-run the program with new arguments, but no changes to the program, you can use `dotnet run 4 fixed --no-build`. Alternatively, you can call the previously built `dll` directly with `dotnet bin/Debug/netcoreapp3.1/StochasticLogistic.dll`. The last approach seemed to run the program with the least lag time. It's also easy to create cross-platform executables of your program (described [here](https://solarianprogrammer.com/2018/08/13/create-net-core-fsharp-app-generate-executables-multiple-operating-systems/)).

Next, we want to run each `logmod` function many times. This `repeat` function repeats an anonymous function, `f`, `n` times. The anonymous function is called with `f()` and the output is cons'd onto the accumulator.

```
    let repeat n f =
        let rec loop n acc =
            match n with
            | 0 -> acc
            | _ -> loop (n - 1) (f() :: acc)
        loop n []
```

Because I wanted to conduct some informal performance comparisons among the functions, I added a couple of more arguments to the top of the program.

```
    let funType = argv.[2]     // "for", "recif", "recmatch", "unfold"
    let reps = int argv.[3]    // number of replications
```

And added this code to the bottom of the program. Because we are not interested in the actual output (only the execution time), the output is piped to `ignore`.

```
    if funType = "for" then
        repeat reps (fun () -> logmodFor t yinit r k thetasd seedType) |> ignore
    elif funType = "recif" then
        repeat reps (fun () -> logmodRecIf t yinit r k thetasd seedType) |> ignore
    elif funType = "recmatch" then
        repeat reps (fun () -> logmodRecMatch t yinit r k thetasd seedType) |> ignore
    elif funType = "unfold" then
        repeat reps (fun () -> logmodUnfold t yinit r k thetasd seedType) |> ignore
    else
        invalidArg "funType" "must be one of for, recif, recmatch, unfold"
```

First, we will run the program with a small number of years and reps to compile it.

```
dotnet run 4 random for 2
```

And then we can do some informal timing tests.  

```
$ time dotnet bin/Debug/netcoreapp3.1/StochasticLogistic.dll 100 random for 50000

real	0m1.355s
user	0m1.280s
sys	0m0.073s

$ time dotnet bin/Debug/netcoreapp3.1/StochasticLogistic.dll 100 random recif 50000

real	0m2.244s
user	0m2.211s
sys	0m0.111s

$ time dotnet bin/Debug/netcoreapp3.1/StochasticLogistic.dll 100 random recmatch 50000

real	0m2.237s
user	0m2.231s
sys	0m0.108s

$ time dotnet bin/Debug/netcoreapp3.1/StochasticLogistic.dll 100 random unfold 50000

real	0m0.516s
user	0m0.492s
sys	0m0.028s
```

I was not surprised to see that using arrays in a `for` loop was nearly 2x as fast as recursion. But I was surprised that using sequences with `unfold` was over 2x as fast as the `for`. I'm wondering if sequences are so lazy that this code is only timing the creation of a lists of sequences to be evaluated later. When I explicitly convert the sequence to a list in `logmodUnfold`, 

```
        Seq.unfold calc y |> Seq.take t |> List.ofSeq 
```

the function is 2x as slow as recursion. 

```
$ time dotnet bin/Debug/netcoreapp3.1/StochasticLogistic.dll 100 random unfold 50000

real	0m4.049s
user	0m4.055s
sys	0m0.185s
```

I don't know how much of the extra time is related to list conversion versus executing the lazy sequences. When I learn more about sequences, I will update this post with an answer.