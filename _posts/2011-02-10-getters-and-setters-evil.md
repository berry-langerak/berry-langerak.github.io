---
layout: post
title: Getters and setters; evil or necessary evil?
---

I’ve been reading a lot about getters and setters over the last few years, including Alan Holub’s
infamous article “Getters and setters are evil“, and Martin Fowler’s article “GetterEradicator.”
Although I do still feel like getters and setters are to be avoided most of the time, it’s hard
to tell where to use them and where you shouldn’t.

I’ve been working on the website of PFZ.nl, the largest PHP community in the Netherlands, together
with about eight other programmers I can’t help but respect. While writing the code for this website,
I started to notice that each and every class in the system had accessors for most, if not all,
protected fields. I’ve started a discussion with my fellow developers, and this is what I’ve argued.

First, I wrote an article on it, which was rather ill-received by some. This is understandable,
as the presence of accessors is deeply rooted in OO PHP code, although this isn’t limited to one
specific language. Besides, my point of view sounded rather puristic in my first article. Before
I want to discuss why accessors should be handled with care, I’d like to go back half a century,
to the introduction of OO.

Back in the day, some great minds thought up the concept of OO to make sure software would be
easier to change, which means it’s easier to maintain. To accomplish that, they thought up the
concept of an Object: a small, focused unit of code that is only ever responsible of one thing.
One of the key concepts of OO was that objects should hide their implementation, to let their
collaborators send them messages that they could handle for themselves. This concept became
known under the term “Encapsulation”, a reinforcement of the guideline “Tell, don’t ask.”

Wikipedia states that the term encapsulation is used to refer to one of two notions: either
to restrict access to an objects’ internals, or a language construct that facilitates the
bundling of data, and methods operating on said data (which I like to call behaviour). Since
the arrival of (decent) OO functionality in PHP, I have seen a lot of accessors which don’t
adhere to the above concept: they actually willingly expose implementation, instead of hiding
it.

This is not necessarily a bad thing, and sometimes, this even makes sense. The problem doesn’t
actually lie with exposing the implementation and the data, but with the collaborator that
*directly* uses that data. The number one example of misusing exposed data would be a bank
account object: you can either opt for having setBalance( $balance ) and getBalance( ) methods,
but common sense would dictate different methods: withdraw, deposit, and transfer. The number
one reason for these methods is that the collaborator knows *nothing* about what a balance is,
nor should it have to.

Translating this example to (PHP) code, the first option with accessors would probably result
into the following class and collaborator.

{% highlight php %}
<?php
class BankAccount
{
	protected $balance;
	public function setBalance($balance)
	{
		$this->balance = $balance;
		return $this;
	}
 
	public function getBalance()
	{
		return $this->balance;
	}
}
 
class Transaction
{
	public function transfer($amount, BankAccount $from, BackAccount $to)
	{
		if(($from->getBalance( ) - $amount) < 0) {
			throw new InsuffientFunds( );
		}

		$to->setBalance($to->getBalance() + $amount);
		$from->setBalance($from->getBalance() - $amount);
		return true;
	}
}
{% endhighlight %}

Now, although this code is easy to write, there might just as well not be a BankAccount object,
as it actually doesn’t *do* anything. It’s devoid of behaviour. This means that it’s a useless
object which might just as well be nuked. Refactoring to the second option (withdraw, transfer,
deposit), the resulting code might be something like this:

{% highlight php %}
<?php
class BankAccount
{
	protected $balance;
	public function tranfer($amount, BankAccount $to)
	{
		$this->withdraw($amount);
		$to->deposit($amount);
	}
 
	public function withdraw($amount)
	{
		if(($this->balance - $amount) < 0) {
			throw new InsufficientFunds();
		}

		$this->balance -= $amount;
		return;
	}
 
	public function deposit($amount)
	{
		$this->balance += $amount;
		return;
	}
}
 
class Transaction
{
	public function transfer($amount, BankAccount $to, BankAccount $from)
	{
		return $from->transfer($amount, $to);
	}
}
{% endhighlight %}

Now, this code (in my opinion) is just as easy to read and write, but there’s a very large
benefits using the second approach: the ability to change. Imagine, for example, that there’s a
“premium” bank account that allows to go in debt to, say, a thousand euros. You’ll have to
change both examples in this case, but you’ll see that the second will accommodate this change
much easier. Let’s refactor both examples, starting with the one that uses accessors:

{% highlight php %}
<?php
class BankAccount
{
    protected $balance;
    protected $overdraw = 0;
 
    public function __construct($balance, $overdraw = 0)
    {
        $this->balance = $balance;
        $this->overdraw = $overdraw;
    }
 
    public function setBalance($balance)
    {
        $this->balance = $balance;
        return $this;
    }
 
    public function getBalance()
    {
        return $this->balance;
    }
 
    public function getOverdraw()
    {
        return $this->overdraw;
    }
}
 
class Transaction
{
    public function transfer($amount, BankAccount $from, BackAccount $to)
    {
        if(($from->getBalance() + $from->getOverdraw() - $amount) < 0) {
            throw new InsuffientFunds();
        }

        $to->setBalance($to->getBalance() + $amount);
        $from->setBalance($from->getBalance() - $amount);
        return true;
    }
}
{% endhighlight %}

Although this works, it’s far from elegant. If you have more than one collaborator for the bank
account object, and you usually do, you’ll have to duplicate that logic to all other collaborators.
If requirements change, you’ll have to change the logic on each place you’re calculating the
balance. There are more issues with the code, but I’ll get to that later. Now, to refactor the
well encapsulated version of the code.

{% highlight php %}
<?php
class BankAccount
{
    protected $balance;
    protected $overdraw = 0;
 
    public function tranfer($amount, BankAccount $to)
    {
        $this->withdraw($amount);
        $to->deposit($amount);
    }
 
    public function withdraw($amount)
    {
        if(($this->balance + $this->overdraw - $amount) < 0) {
            throw new InsufficientFunds();
        }

        $this->balance -= $amount;
        return;
    }
 
    public function deposit($amount)
    {
        $this->balance += $amount;
        return;
    }
}
 
class PremiumBankAccount extends BankAccount
{
    protected $overdraw = 1000;
}
 
class Transaction
{
    public function transfer($amount, BankAccount $to, BankAccount $from) {
        return $from->transfer($amount, $to);
    }
}
{% endhighlight %}

Alright, the introduction of the new class PremiumBankAccount could have been
done with the setters and getters too, but still, the change had just one line.
Better yet: the change only affected the BankAccount class itself, and the rest
of the application is still blissfully unaware of any change, although the
functionality has been radically changed. The collaborator doesn’t know the
bank account even can overdraw, nor should it have to.

If you’re still not convinced the second example is better, I’ll play devils
advocate and I’ll ask you to refactor both examples to incorporate
functionality so that a bank account can have a currency, say EUR and USD, and
you should be able to transfer an arbitrary amount of dollars to a bank account
based on EUR. You’ll see how much your Transfer class will have to do, when
you’re using only accessors.

Alright, I think I’ve made my point by now: the second example is much better,
and indeed, it’s OO like OO was meant to be. Now, obviously, you wouldn’t do it
like I did it in the first example, right? You’d *maybe* write the accessors,
but you’d put the transfer logic into the bank account itself, right? Good for
you, but most classes I see out there today actually lean toward the first
example, and this yields my curiosity in finding out: why?

Just open an OO library written in PHP, and grep for “function set”, and you’ll
be shocked. Again, the problem doesn’t lie with the accessor per se, but you
should check the collaborators, and I can guarantee you that the collaborators
are doing calculations and making decisions which should belong in the object
itself. By exposing the data, you’re making this possible.

If you work with others you can simply expect people to change your objects’
behaviour instead of using the accessors you wrote, or you can provide well
named methods that encapsulate the data, so your coworkers know how to use your
class.

So, why do we use so much setters and getters if there is seemingly no need to
do so? Well, first of all, I’d like to go back a decade, when I was still in
school and getting lessons on programming in Java. Indeed, my books were
full of getters and setters, so I just assumed this was the way things
work. How could I have known better? The people that wrote the books are
notably smarter than I am, so they must be right. Right? Well, not so much.

People starting to learn OO are reading books, tutorials and example code which
are littered with accessors. It just makes good sense to copy that pattern onto
your own code. If, like me, you were used to writing procedural code, you’ll
even feel right at home. You still manipulate the variables, only now they’re
in some sort of namespaced array called an object. Awesome. I hereby bid people
that write introductory tutorials to stop using accessors “because they can”,
and instead only write them “because they have to”.

So, “learning it the wrong way” is a big reason, but there’s another one, and
that is layering. Layering your application is supposed to make your code more
maintainable, more extendable, more reusable, and more of that. In PHP, there
is not a single framework that doesn’t claim to “be MVC”, and that is another
reason for the presence of accessors everywhere.

If you split up responsibilities into “business logic” (such as transferring an
amount to another bank account), and the logic for displaying stuff (such as,
when the accounts’ balance is negative, display the balance in red), you’re
supposed to be able to change one layer without changing the other.
Implementing the separation between those responsibilities will make it easier
to display one object in multiple ways.

For example, say I want to display the bank accounts in HTML. I’ve got a few
options, one of them being rendering the HTML inside the object itself. Once
you start down that road though, your BankAccount class will grow to extreme
proportions, as you might want to display the information in HTML, XML, JSON,
etc., etc.

Another option is to leave the displaying of the bank account to a template.
This template will fetch the accounts’ balance and if < 0, display the balance
in red for HTML. Indeed, this is the most widely used technique and, in and on
itself, not a bad one to start with. It does have a requirement though: each
part of data that you want to display will have to be available to the View
layer. As PHP doesn’t know “Friend” classes like C++, and there’s no ability to
read protected variables from the same “package” like Java, you’ll have to make
the access public. This means that you’ll have to write a public getter for
each value you want to display!

On the other side of the object, there’s storage. Generally, objects from the
model layer will have to be saved somehow, and that yields the exact same
issue: how does the object that saves the values (say, a DataMapper) get the
values from the target object? In the exact reverse, how does a DataMapper
build the object without being able to set values? You could use the
constructor for this, but that might mean you get a constructor with 10+
parameters, which is not the most clean solution ever.

All in all, I don’t think accessors are evil per se, but you should only ever
use the accessors in cross-layer situations. Don’t use accessors to build
functionality, but only because you need the value to display and/or save.
