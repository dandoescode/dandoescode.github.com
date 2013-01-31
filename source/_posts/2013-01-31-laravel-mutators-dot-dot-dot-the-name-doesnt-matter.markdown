---
layout: post
title: "Laravel 4 Mutators...The Name Doesn't Matter"
date: 2013-01-31 01:45
comments: true
categories:
---

There was a change to the [Laravel 4](http://four.laravel.com) Eloquent Mutators recently that some people did not like.  The thing that changed was the prefix for the Mutator methods.  So, instead of naming them `getFoo` and/or `setFoo` you name then `giveFoo` and `takeFoo`.  Some people seemed to think that this would impact how they actually used their Models, which it does not.  It actually makes it easier for you to write them.

## What is an Eloquent Mutator?

So the Mutators are simply methods on your Model that you can define to modify the value of a field either before it is returned to you, or prior to being set.  These methods are called automatically when you access or set the field as a property on the model (e.g. `$user->settings`).

Let's take that `settings` for example. For this we will assume that `settings` is a field that holds a JSON encoded array of settings. Here is a basic User model with using Mutators to parse the permissions:

``` php
<?php
class User extends Illuminate\Database\Eloquent\Model
{
    protected function giveSettings($settings)
    {
        return json_decode($settings);
    }

    protected function takeSettings($settings)
    {
        return json_encode($settings);
    }
}
```

Notice one important thing here, both methods are `protected`.  This is because these methods should never be called from outside the scope of the Model. This is not required, but it is a good habit to get into.  Now to access the permissions you just do `$user->settings`.

## Why not 'get' and 'set'?

There are multiple reasons:

1.  Naming conflicts with the Eloquent core.  There are methods such as `getTable` defined, which meant that you could never have a field named `table`, or it would fall over (well, return incorrect data in this simple case).
2.  You should be able to create your own `getFoo` and `setFoo` methods without having to worry about conflicting with anything (including future fields) or having to worry about the method prototype matching what Eloquent wants.

Number 1 could be fixed by just changing the Model methods to stop using the `getFoo` scheme, but that would cause people just as much "frustration", if not more than changing the mutator naming.

Number 2 is the bigger issue.

### Real World Example

Going to use one of the examples that [@ben_corlett](http://twitter.com/ben_corlett) used, as it makes a lot of sense.

So, to make things nice and DI friendly (there are other reasons to do it as well), you can use Interfaces.  So let's define a simple one for a User.

``` php
<?php
interface UserInterface
{
    public function getName();
}
```

So here would be most people's (including my own) first attempt at implementing the interface:

``` php
<?php
class User extends Illuminate\Database\Eloquent\Model implements UserInterface
{
    public function getName()
    {
        return $this->name;
    }
}
```

With the old mutator naming, this would be a mutator and cause an infinite loop. With the new naming, this is no longer a mutator and works as expected.

### Conclusion

Are `give` and `take` the best prefixes? No. Are they horrible? No. Method naming is not a "one size fits all" type thing, some people will like them, and some people will hate them. This is one of those changes that everyone gets up in arms about, and then in a few days, your code will be changed and

Here is a little anecdote that pertains to this: When I was creating FuelPHP, I changed all the `factory` methods in the entire framework to `forge`, and everyone freaked out.  A week later, everyone got used to it and went back to writing code.


