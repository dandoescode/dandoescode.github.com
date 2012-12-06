---
layout: post
title: "Is Eval Really Evil? Yes...and No."
date: 2012-08-28 00:00
comments: true
categories: [PHP]
---

How many times have you been told "Eval is Evil"?  Probably a lot, and for good reason.  The `eval()` function in PHP is dangerous in most situations.  Why?  Because executing arbitrary code can lead to very bad things.  Let's look at a real world example that I have seen used in the wild (not going to say which app...for reasons you will see).

The offending code looked something like this (class/variable names have been changed):

``` php
<?php
$method = $_GET['method'];
eval('$return = Foo::'.$method.'();');
?>
```

_If you look at that code and say "What is wrong with that?", then you may need to re-evaluate your career path, or read on and learn._ 

The above code was used to be able to call various methods (obviously).  However, this is the worst possibly way to do this.  It is very easy to exploit.  All the user needs to do is find a valid method, then they can execute almost any code they want (let's assume this was in index.php):

```
index.php?method=bar()%3B%24r%3Dmysql_query(%22select%20*%20from%20users%22)%3Bwhile(%24row%3Dmysql_fetch_assoc(%24r))var_dump(%24row)%3Bdie()%3B%2F%2F
```

If you decode that you will see that method is set to:

```
bar();$r=mysql_query("select * from users");while($row=mysql_fetch_assoc($r))var_dump($row);die();//
```

So when the original code is executed it will run the following:

``` php
<?php
eval('$return = Foo::bar();$r=mysql_query("select * from users");while($row=mysql_fetch_assoc($r))var_dump($row);die();//();');
?>
```

So, this will (assuming that `mysql_connect()` has been called and a DB selected) basically dump the entire `users` table to that user.  I hope you can see why THAT is bad.

This is probably enough to turn you away from using it at all...which isn't a bad thing.  However, this is where I will get hate, there are times where you have no other option to use `eval()`, or the other options are equally as bad.

### Wait, I can use eval()?

Well...sometimes, and only if you are very, VERY, careful.  One of the only valid reasons I can think of to use `eval()` is Integration Tests.

The main reason that it is OK to use it when writing Integration Tests is because you control everything.  You control the code being executed, and the environment in which it is executed.  The tests deal with no end user data, so you don't need to worry (unless you have some evil devs on your team) about arbitrary code execution.

A great example of this use is reading in tests from external files.  These files can contain various data besides the code to execute to run the tests.  So you read in the file, get the code that needs to be run and run it through `eval()` (you could also write the code out to a temp file, then `include` it, but it poses the same security risks, and adds unneeded load to the filesystem).

### Ok, so only in tests?

Knowing when to use `eval()` is very simple.  Ask yourself "Will this code run in production with user input?" If your answer is yes, then NO, you should not use `eval()`...ever (unless you REALLY know what you are doing and are REALLY sure the code being executed is safe).

### Catching Syntax Errors in Eval

Since errors triggered in code ran inside `eval()` will not be caught by your error handler (set via `set_error_handler()`), you need to implement some trickery to catch PHP errors.  This can be handy when you are using `eval()` to execute some test code.

Here is the full code sample for catching the errors:

```php
<?php
// We need to make sure we don't have any previous errors munging our results
@trigger_error('');
eval($code);
$error = error_get_last();

// If the message is empty, then we know it is from our error triggered above
if ($error['message'] !== '') {
    // Do whatever you want with the error...here we throw an ErrorException
    throw new ErrorException($error['message'], $error['type']);
}
?>
```

Ok, before you start hating on the code above, it is meant only for use during testing, so using the `@` operator isn't a big deal (c'mon, we are using `eval()` anyway).  As you can see, there is not that much to it: Just trigger a blank error, execute the code, check to see if a new error was triggered.

I am using this in some Integration Tests I am writing for Lex for testing templates and such.  I add a little more "magic" to add the correct line number in the test file that the error occured.  This is possible because in the above code `$error['line']` is equal to the line number of the evaluated code in which the error occured.

### Haters Gonna Hate

I know I am going to get hate for this article, but that is OK.  Most of you are probably going to hate just because you have heard the "Eval is Evil" line your entire career, so you have been brainwashed into thinking a certain way.  Don't hate on something just because it has a bad rep (`goto` anyone?).

