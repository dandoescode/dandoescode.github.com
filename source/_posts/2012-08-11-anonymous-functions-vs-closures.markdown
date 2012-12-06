---
layout: post
title: "Anonymous Functions vs. Closures"
date: 2012-08-11 00:00
comments: true
categories: [PHP]
---

Ever wonder what the difference between an Anonymous Function and a Closure is?  What about a Lambda?  How are they implemented in PHP?  There seems to be a lot of confusion in this area, with people interchanging the words, and not really knowing what they mean, so lets dive in and answer these questions.

First things first: A "Lambda" is just another name for an Anonymous Function.  You may have seen it used in some programming language (e.g. Python) and not known what it was...now you do.

Now that the easy stuff is out of the way, lets take a look at what these two things _actually are_.  If you are confused at first, don't worry, I will hopefully clear it up for you in a bit, these are just brief definitions of the two.

**Anonymous Function** - An Anonymous Function is a function that is defined, and occasionally invoked, without being bound to an identifier.  It also has the variable scoping characteristics of a Closure (see below).

**Closure** - A Closure is a function that captures the current containing scope, then provides access to that scope when the Closure is invoked.  The Closure binds non-local variable references to the corresponding variables in scope at the time the Closure is created.

From this we can derive: All Anonymous Functions are Closures, but not all Closures are Anonymous Functions.

### Layman's Terms

An Anonymous Function is a Closure without a name.  A Closure is a function which binds references to non-local variables to local variables inside the function at the time of the Closure definition. 

### JavaScript Examples

Here a few examples written in JavaScript (I used JS here since most people have experience in using is):

**Anonymous Function as Callback**

``` javascript
function getPosts(callback) {
    // Code to fetch posts would go here...
    if (response.status == 200) {
        callback(response.body);
    }
}

getPosts(function (posts) {
    // Do something with the posts here
});
```

**Closure as Return Value**

``` javascript
function getProcessor(driver) {
    // Do some magic to load the processing driver here...

    // Imagine we set the driver variable to the
    // processing driver object here...

    return function (data) {
        // Notice we have access to the driver var. from the
        // getProcessor() scope.
        return driver.process(data);
    }
}

var proc = getProcessor('json');

console.log(proc(someJsonDataHere));
```

### How about PHP?

Here is where things get interesting.  In PHP 5.3, Closures and Anonymous Functions were added to the language.  However, they have their quirks (one of which was fixed in PHP 5.4).

Anonymous Functions are implemented as `Closure` objects.  PHP takes the idea that "Anonymous Functions are Closures without a name" to heart, because that is EXACTLY what they are.

Here is a simple Anonymous Function being sent to the array_map function to multiply all integers in an array by 2 (useless, I know, but hey, it is just an example):

``` php
<?php
$arr = array_map(function ($val) {
    return is_int($val) ? $val * 2 : $val;
}, $arr);
?>
```

Simple, right?  Now ask yourself this: What if you want to access non-local variables in that function?  After all Anonymous Functions are Closures too.  Well, you can, but there is a catch:  You have to explicitly tell PHP which variables it should use. That is the first of a few quirks, so let us dive into them.

#### Quirk #1

You MUST send the function all of the variables you want bound to the scope of the Closure using the `use` keyword.  This is different from most other languages, where this is done automatically (like in my JS example above).

``` php
<?php
$foo = 'foo';
$bar = 'bar';

$baz = function () use ($foo, $bar) {
    echo $foo, $bar;
};
?>
```

#### Quirk #2

The bound variables are **copies**, of the variable, not references.  If you want to be able to change the variable inside the Closure, you MUST send it by reference:

``` php
<?php
$foo = 'foo';
$bar = 'bar';

$baz = function () use (&$foo, &$bar) {
    $foo = 'Hello ';
    $bar = 'World!';
};
$baz();
echo $foo, $bar; // Outputs "Hello World!";
?>
```

#### Quirk #3

In PHP 5.3 if you are using a Closure inside of a class, the Closure will not have access to `$this`.  You must send a reference to `$this`, however, you cannot send `$this` directly:

``` php
<?php
class Test
{
    protected $foo = 'foo';

    public function baz() {
        // Get a reference to $this
        $self = &$this;
        $func = function () use ($self) {
            echo $self->foo;
        }
    }
}
?>
```

*Note: Because of the way object references are handled in PHP, you do not have to send a reference to `$self` if you wish to modify it.*

### PHP 5.4 to the Rescue

In PHP 5.4, they have added support for the usage of `$this` in Closures.  They do this by binding the object to the Closure at the time of definition.

You can also change which object your Closure is bound to by using the `bind()` and `bindTo()` methods.  You can read more about how to use these methods and exactly what they do in the [PHP documentation](http://php.net/manual/en/class.closure.php).

### So...

How should you know when to call it a Closure or Anonymous Function?  Simple:  You can always call it a Closure (because all Anonymous Functions are Closures), and if it doesn't have a name, it is Anonymous.  Pretty simple, right?

Hope that helped clear up the confusion.  Feel free to leave questions in the comments, or I am always available on [Twitter](http://twitter.com/dhorrigan).
