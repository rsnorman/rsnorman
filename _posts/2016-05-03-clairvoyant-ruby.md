---
layout: post
title:  "Clairvoyant Ruby"
date:   2016-03-03 08:00:00 -0500
author: "Ryan Norman"
tags:   "override, developer, programming, coding, rails, ruby, module, dynamic programming"
excerpt: "I'm often told it's impossible to predict the future but does that same axiom hold true for overriding methods yet to be defined in Ruby?"
---

The great thing about programming is it always finds a way to humble you.

As I spend longer in the field, I **should** know better than to think something
that once was complicated for me is now magically easy because I've done it a
thousand times. Emphasis on "should" if the bolding wasn't obvious enough.

For example, I recently started working on a library to mark and safely expose
personal data. My thought process is to mark a field as safe to display similar
to how Rails used to allow html in ERB. The one caveat is the method must
be in a block that allows this unwrapping. No problem, right?

So, I create a nice mixin for controllers that allows me to mark actions as
safely exposing person information.

{% highlight ruby %}
# app/controllers/persons_controller.rb
class PeopleController < ApplicationController
  safely_expose :show

  def show
    @person = Person.find(params[:id])
  end
end
{% endhighlight %}

{% highlight html %}
# app/views/people/show.html.erb
<h1><%= @person.first_name.expose! %></h1>
{% endhighlight %}

Cool, no problem. I take advantage of Rails `around_action` callback
and this works great.

{% highlight ruby %}
module SafeExpose
  def self.included(base)
    class << base
      def safely_expose(*actions)
        around_action :safe_expose, only: actions
      end
    end
  end

  def safe_expose
    SafeExpose.expose do
      yield
    end
  end
end
{% endhighlight %}

I'm feeling good. My ten years of experience is paying off.

*We all know it's going to take a turn for the worse now.*

Next up is the same treatment for our Sidekiq jobs that send personal data to
third-parties. The JSON also must be serialized safely and the jobs marked as
allowing this safe action. I know right off the bat that Sidekiq workers don't
have nice callbacks like Rails controllers. But we won't let a little thing like
this hold us back. I'll just override the `perform` method that each worker
implements.

{% highlight ruby %}
module SafeExpose::Sidekiq
  def perform(*args)
    SafeExpose.expose do
      super(args)
    end
  end
end
{% endhighlight %}

I confidently add this to my first Sidekiq worker&hellip;
{% highlight ruby %}
class SendPersonService
  include Sidekiq::Worker
  include SafeExpose::Sidekiq

  def perform(person_id)
    http_post(first_name: Person.find(person_id).first_name.expose)
  end
end
{% endhighlight %}
&hellip;and it fails?

Okay, yeah, of course, my `SendPersonService` is actually inheriting from my
`SafeExpose::Sidekiq` module and `Sidekiq::Worker` classes are just duck types.

Well, fiddlesticks. I start to think I'm stuck until realizing I can use that
Rails monkey-patched method `alias_method_chain` that I've honestly never called
correctly without looking at the documentation. Let's take a crack at using it.

{% highlight ruby %}
module SafeExpose::Sidekiq
  def self.included(base)
    class << base
      alias_method_chain :perform, :exposure
    end
  end

  def perform_with_exposure(*args)
    SafeExpose.expose do
      perform_without_exposure(*args)
    end
  end
end
{% endhighlight %}

Looking good. I no longer have to worry about inheritance but I do know one of
the quirks with `alias_method_chain` is it must be called after the method is
defined. Which means I have to include my module after `perform`.

{% highlight ruby %}
class SendPersonService
  include Sidekiq::Worker

  def perform(person_id)
    person = Person.find(person_id)
    http_post(first_name: person.first_name.expose!)
  end
  include SafeExpose::Sidekiq
end
{% endhighlight %}

Ugh, that's ugly *and* unintuitive.

What else can I do? Do I have any other tricks up my sleeve that don't
require including my mixin after the method. I dig real deep, like past the elbow.
I remember reading once about a new way to add modules to a Ruby class called
`prepend`. It ends up inserting the module after the class in the inheritance
hierarchy. The class it is added to actually inherits from the
module. A light bulb turns on and I realize if I can have `SendPersonService`
inherit from my `SafeExpose` module, I can once again override the `perform` method:

{% highlight ruby %}
class SendPersonService
  include Sidekiq::Worker
  prepend SafeExpose::Sidekiq

  def perform(person_id)
    http_post(first_name: Person.find(person_id).first_name.expose)
  end
end
{% endhighlight %}

Whammy!!! We can now use our module and expose personal data safely in our
Sidekiq job.

*Buttttt*&hellip; this still bugs me. Really, `prepend`? It's been around since
Ruby 2.0 but not many encounter or use it in their daily Ruby or Rails work. I
can just see the questions (accusations) of why the `SafeExpose::Sidekiq` is
not working when included.

**Is there another option?**

I wrack my brain. I try to figure out how to hook into Sidekiq's source code. I
do what we do when presented with a problem we haven't seen before: I search
StackOverflow. Nothing, nada, `nil`. My confidence was sky high upon the
beginning of this endeavor but saying it's been knocked a few pegs is an
understatement.

I'm about to give up and plan on writing documentation on what `prepend` does
and why it must be used when I stumble upon an interesting hit way down the
Google search results. A callback method I've never heard of called
`method_added`. "Naw, that's too easy," I tell myself. I dig into the documentation
like my 8-month pregnant wife treats a bowl of ice cream. Sure enough, it's
possible to know when a method is added to a class in Ruby. The solution comes
to me immediately:

{% highlight ruby %}
module SafeExpose::Sidekiq
  def self.included(base)
    class << base
      def method_added(method)
        return if instance_methods.include?(:perform_without_exposure) # Stop rechaining method
        return unless method == :perform
        alias_method_chain :perform, :exposure
      end
    end
  end

  def perform_with_exposure(*args)
    SafeExpose.expose do
      perform_without_exposure(*args)
    end
  end
end
{% endhighlight %}
&hellip;[and then](https://youtu.be/oqwzuiSy9y0?t=12)&hellip;

{% highlight ruby %}
class SendPersonService
  include Sidekiq::Worker
  include SafeExpose::Sidekiq

  def perform(person_id)
    http_post(first_name: Person.find(person_id).first_name.expose)
  end
end
{% endhighlight %}

The underlying code becomes a bit more complicated but it's use is simple and
best of all, we can override future methods in a way that doesn't require us to
remember less-understood Ruby concepts. I once again don't feel useless&hellip;
well, until the next time my hubris gets the best of me.
