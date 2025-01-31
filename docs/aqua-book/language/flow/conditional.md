# Conditional

Aqua supports branching: you can return one value or another, recover from the error, or check a boolean expression.

## Contract

* The second branch of the conditional operator is executed if and only if the first block failed.
* The second block has no access to the first block's data.
* A conditional block is considered "executed" if and only if any inner block was executed successfully.
* A conditional block is considered "failed" if and only if the second (recovery) block fails to execute.

_Block of code is any number of lines on the same indentation level or deeper_

## Conditional operations

### try

Tries to perform operations, or swallows the error (if there's no catch, otherwise after the try block).

```aqua
try:
   -- If foo fails with an error, execution will continue
   -- You should write your logic in a non-blocking fashion:
   -- If your code below depends on `x`, it may halt as `x` is not resolved.
   -- See Conditional return below for workaround
   x <- foo()
```

### catch

Catches the standard error from `try` block.

```aqua
try:
   foo()
catch e:
   logError(e)
```

Type of `e` is:

```aqua
data LastError:
  instruction: string -- What AIR instruction failed
  msg: string -- Human-readable error message
  peer_id: string -- On what peer the error happened
```

### if

If corresponds to `match`, `mismatch` extension of π-calculus.

```aqua
x = true
if x:
  -- always executed
  foo()

if x == false:
  -- never executed
  bar()

if x != false:
  -- executed
  baz()
```

Currently, you may only use one `==`, `!=` operator in the `if` expression, or compare with true.

Both operands can be variables.

### else

Just the second branch of `if`, in case the condition does not hold.

```aqua
if true:
  foo()
else:
  bar()
```

If you want to set a variable based on condition, see Conditional return.

### otherwise

You may add `otherwise` to provide recovery for any block or expression:

```aqua
x <- foo()
otherwise:
  -- if foo can't be executed, then do bar()
  y <- bar()
```

## Conditional return

In Aqua, functions may have only one return expression, which is very last. And conditional expressions cannot define the same variable:

```aqua
try:
  x <- foo()
otherwise:
  x <- bar() -- Error: name x was already defined in scope, can't compile
```

So to get the value based on condition, we need to use a [writeable collection](../types.md#collection-types).

```aqua
-- result may have 0 or more values of type string, and is writeable
resultBox: *string
try:
  resultBox <- foo()
otherwise:
  resultBox <- bar()

-- now result contains only one value, let's extract it!
result = resultBox!

-- Type of result is string
-- Please note that if there were no writes to resultBox, 
-- the first use of result will halt.
-- So you need to be careful about it and ensure that there's always a value.
```
