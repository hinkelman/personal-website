+++
title = "Proof of Kaprekar's constant in Scheme, Python, and Elixir"
date = 2024-05-30
[taxonomies]
tags = ["Scheme", "Python", "Elixir"]
+++

I thoroughly enjoyed this [delightful post](https://demian.ferrei.ro/blog/programmer-vs-mathematician) on [Kaprekar's constant](https://en.wikipedia.org/wiki/6174), which was new to me. Similar to the post author, I also like to use programming to understand mathematical concepts. I thought it would be a fun exercise to translate the example Ruby code to my favorite language, Scheme, and two other languages that I'm learning, Python and Elixir.

<!-- more -->

Kaprekar's routine is an iterative process that starts with a 4-digit number and (usually) ends at 6174. Here is an example starting with 55:

```
5500 - 0055 = 5445
5544 - 4455 = 1089
9810 - 0189 = 9621
9621 - 1269 = 8352
8532 - 2358 = 6174
7641 - 1467 = 6174
```

In Scheme and Python, the approach is to convert the number to a string, left pad with zeros (if necessary), split into a list of characters (Scheme) or strings (Python), sort the list, convert back to string, and then convert to number.  

Python provides a function for left padding a string.

```
>>> str.rjust("55", 4, "0")
'0055'
```

In Scheme, we use `format` to write a left pad procedure. Format directives are written as strings that start with `~`. The `~a` directive prints the object as with `display`. Commas separate four optional arguments: `mincol`, `colinc`, `minpad`, `padchars`. The width of the output is specified with `mincol`. We use the default values for `colinc` and `minpad`, which is why we need `,,,`. The `padchars` are preceded by `'`. The `@` indicates that the padding is placed on the left. 

```
(define (string-pad-left x mincol padchars)
  (format
   (string-append "~" (number->string mincol) ",,,'" padchars "@a")
   x))

> (string-pad-left "55" 4 "0")
"0055"
```

In Elixir, we are only working with integers; no conversion to characters or strings. We "pad" the list instead of the string. `++` concatenates two lists.

```
def pad_left(lst, width, value) do
  if length(lst) < width do
    List.duplicate(value, width - length(lst)) ++ lst
  else
    lst
  end
end

> pad_left([5, 5], 4, 0)
[0, 0, 5, 5]
```

In Scheme and Python, we create functions that convert a number to a list of characters (Scheme) or strings (Python) and back again. In both cases, we build the 4-digit left padding into the conversion to lists and, thus, `number->list` and `num2list` are not as general as their names imply. `num2list` takes advantage of strings being iterable and uses a list comprehension to pull apart the newly created string (from `n`) into a list. The `join` syntax is a bit weird to me as a Python novice. Elixir provides `digits` and `undigits` that provide the functionality we need without working with strings.

```
;; Scheme
(define (number->list n)
  (string->list (string-pad-left (number->string n) 4 "0")))

(define (list->number lst)
  (string->number (list->string lst)))

# Python
def num2list(n):
    return [x for x in str.rjust(str(n), 4, "0")]

def list2num(lst):
    return int("".join(str(x) for x in lst))

# Elixir
> Integer.digits(1234)
[1, 2, 3, 4]
> Integer.undigits([1, 2, 3, 4])
1234
```

For repdigits (e.g., 1111, 2222), the result of the Kaprekar routine is 0, not 6174. Although not strictly necessary, I will follow the original post and create functions for detecting repdigits. The Python and Elixir functions use the same logic but different approachs, i.e., strings and sets in Python and unique list values in Elixir. The Scheme version iterates over all characters (after converting the number to list of characters) to check if each character is the same as the first. Scheme and Elixir both have the convention of using `?` in boolean functions, which I think pops out nicely when reading code (arguably better than the `is_` prefix).

```
;; Scheme
(define (repdigit? n)
  (let ([lst (number->list n)])
    (for-all (lambda (x) (char=? x (car lst))) lst)))

# Python
def is_repdigits(n):
    return len(set(str(n))) == 1

# Elixir
def repdigits?(n) do
  length(Enum.uniq(Integer.digits(n))) == 1
end
```

As pointed out in the original post, for the Kaprekar routine, we are only working with 4-digit numbers, which allows for a simpler approach using modulo.

```
;; Scheme
(define (repdigit? n)
  (= (modulo n 1111) 0))

# Python
def is_repdigits(n):
    return n % 1111 == 0

# Elixir
def repdigits?(n) do
  Integer.mod(n, 1111) == 0
end
```

Our functions for sorting digits allow for specifying sort direction (via `pred`, `rev`, and `sorter`). As a reminder, padding is included in `number->list` and `num2list`, but first appears for Elixir in `sort_digits`.

```
;; Scheme
(define (sort-digits n pred)
  (list->number (sort pred (number->list n))))

# Python
def sort_digits(n, rev):
    return list2num(sorted(num2list(n), reverse = rev))

# Elixir
def sort_digits(n, sorter) do
  Integer.undigits(Enum.sort(pad_left(Integer.digits(n), 4, 0), sorter))
end
```

We use recursion to implement the Kaprekar routine, which is similar in all three languages. None of these functions have guards against numbers bigger than four digits so you can end up in an infinte loop.

```
;; Scheme
(define (kap n)
  (let ([d (- (sort-digits n char>?) (sort-digits n char<?))])
    (if (= n d) n (kap d))))

# Python
def kap(n):
    d = sort_digits(n, True) - sort_digits(n, False)
    if (n == d):
        result = n
    else:
        result = kap(d)
    return result

# Elixir
def kap(n) do
  d = sort_digits(n, :desc) - sort_digits(n, :asc)
  if n == d, do: n, else: kap(d)
end
```

The last step is to iterate through all integers from 0 to 9999 and raise an exception if the routine doesn't result in 6174 (or is a repdigit). These blocks of code all execute silently and provide proof of Kaprekar's constant. Note that `repdigits?(n)` can be replaced by `kap(n) == 0` (or similar for the other languages).

```
;; Scheme
(for-each
 (lambda (n)
   (unless (or (repdigit? n) (= (kap n) 6174))
     (assertion-violation
      "kap loop"
      (string-append
       (number->string n)
       " is not a repdigit nor does it converge to 674"))))
  (iota 10000))

# Python
for n in range(0, 10000):
    if not (is_repdigits(n) or kap(n) == 6174):
        raise Exception(str(n) + " is not a repdigit nor does it converge to 674")

# Elixir
Enum.each(
  0..9999,
  fn n ->
    unless repdigits?(n) or kap(n) == 6174 do
      raise("#{n} is not a repdigit nor does it converge to 674")
    end
  end
)
```




