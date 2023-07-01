+++
title = "Guess the number game in Fortran"
date = 2021-04-07
[taxonomies]
categories = ["Fortran"]
tags = ["guess-number", "random-variates"]
+++

I recently came across [this blog post](https://opensource.com/article/21/4/compare-programming-languages) on writing simple test programs in different programming languages as a way to get a feel for a language. At the bottom of the post, there were links to articles on implementations of a number guessing game in 13 different languages, [including Fortran](https://opensource.com/article/21/1/fortran). I was curious about the Fortran example because I've been interested in learning a little Fortran after hearing about efforts to improve the tooling around Fortran (e.g., [here](https://ondrejcertik.com/blog/2021/03/resurrecting-fortran/) and [here](https://youtu.be/JUHS-JFvs90)). Well, the Fortran example was written in Fortran 77 so I decided that re-writing it in modern Fortran would be a nice little exercise.

<!-- more -->

I'm not going to write about the Fortran 77 version. It is well described in the [other article](https://opensource.com/article/21/1/fortran). Before I show the code for the "modern" version, I will introduce a couple of subroutines that I plucked from [this random number tutorial](https://masuday.github.io/fortran_tutorial/random.html). 

```
subroutine random_stduniform(u)
   implicit none
   real,intent(out) :: u
   real :: r
   call random_number(r)
   u = 1 - r
end subroutine random_stduniform

! assuming a<b
subroutine random_uniform(a,b,x)
   implicit none
   real,intent(in) :: a,b
   real,intent(out) :: x
   real :: u
   call random_stduniform(u)
   x = (b - a) * u + a
end subroutine random_uniform
```

The first thing to note is that Fortran has both subroutines and functions. I have only a fuzzy understanding of when you would choose one over the other. In this case, it seems that `random_stduniform` and `random_uniform` could have also been written as functions (but I didn't try). Perhaps the fact that `random_number` is a subroutine makes it more intuitive to build subroutines on top of it. When calling a subroutine, you need to preface the statement with `call`, which is not required for functions. Also, in a subroutine, assignment is handled by passing the variable name as an argument. For example, in `random_stduniform`, the variable `r` is declared with type `real` and then assigned a value when passed to the `random_normal` subroutine. 

One of the other unusual (to me) components of these subroutines is `implicit none`, which means that the types of variables need to be explicitly declared and not implicitly inferred from the variable names. Here is an [argument](https://medium.com/modern-fortran/implicit-none-and-carry-on-860a1a0f143b) for embracing `implicit none` rather than seeking to change it in a future version of the Fortran Standard.

Armed with those two sub-routines, here is the full program for the number guessing game:

```
program guessnum
  implicit none

  real :: number
  integer :: number_int, guess

  call random_seed()
  call random_uniform(0.999, 100.0, number)
  number_int = int(number)
  guess = 0

  print *, "Guess a number between 1 and 100"

  do while (guess /= number_int) 
      read *, guess
      if (guess < number_int) then
        print *, "too low"
      else if (guess > number_int) then
        print *, "too high"
      end if
  end do 

  print *, "correct!"
 
end program guessnum
```

After declaring the types of our variables, we set a random seed and generate a number from a random uniform distribution that is > 0.999 and <= 100. `random_uniform` assigns a real number to the variable `number` so we coerce it to an integer with `int`, which has the same effect as `floor` or `truncate` operators in other languages. `guess` is initialized at zero to get us into our `while` loop because `number_int` should never be equal to zero in this program.

We print the guessing instructions, enter the `while` loop, and read the input from the user. Then, we compare the user's `guess` to `number_int`, give feedback, and read their next guess. When `guess` matches `number_int`, we break out of the `while` loop and print `correct!` before exiting the program. 

If you have the GNU Fortran compiler installed, and the code above in a file called `guess.f90`, you can compile the program with `gfortran guess.f90 -o guess` and run the program with `./guess`. Here is example output from running the program:

```
 Guess a number between 1 and 100
50
 too high
25
 too low
37
 too low
43
 too high
40
 too high
39
 too high
38
 correct!
```

That was a fun first experience with Fortran. I'm happy to trade some verbosity for simplicity. I think the Fortran example is a lot easier to understand than the [Rust example](https://opensource.com/article/20/12/learn-rust). Despite working for many years as a scientific programmer (but never officially my job title), I had never considered learning Fortran. I assumed it was an ugly relict hanging on in legacy codebases, but I no longer think it is ugly and it even moved back into the top 20 of the [Tiobe index](https://www.tiobe.com/tiobe-index/).