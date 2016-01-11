---
layout: post
title:  "Raising Errors That Make You Feel Good"
date:   2016-01-11 08:00:00 -0500
author: "Ryan Norman"
tags:   "ruby, rails, exceptions, errors, developer, programming, coding"
---
Sometimes I like seeing errors. You heard me right, seeing an issue come through
in Sentry, Rollbar, Airbrake or whatever floats your boat force our team to
improve our product and that opportunity gives me the warm-fuzzies.

You know what doesn't make me happy, generic errors that give me nothing. How
many times have you received an `ActiveRecord::RecordNotFound` exception and had
no idea what was actually missing? Sometimes it's obvious from the URL and others
it takes some digging. No matter what, it's not immediately clear.

We have this issue happen often with tokens that are passed around our different
services for identifying accounts. Someone made the correct decision to raise
an error when the account could not be found:

{% highlight ruby %}
  def account(account_token)
    Account.find_by!(token: account_token)
  end
{% endhighlight %}

This is good because sometimes an account doesn't get set up correctly or the
token was pasted in with a missing character. I'll even admit that it could be
a programmer error and no token is even sent in. Want to take a guess at the
wonderful error we receive:

{% highlight ruby %}
  ActiveRecord::RecordNotFound: ActiveRecord::RecordNotFound
{% endhighlight %}

I can figure out from the stack trace what's going on. Most of the devs who have
been on the product for a bit know right away too. This is only because we've seen
it before. Even then we aren't sure which token is being sent in since our error
service can only record so much POST data.

This is a rare but not rare enough error that we see that always takes too much
research. *Why are we making this so difficult on ourselves?*

Let's take the most simple step and at least give more information
when raising the `ActiveRecord::RecordNotFound` exception.

{% highlight ruby %}
  def account(account_token)
    Account.find_by!(token: account_token)
  rescue ActiveRecord::RecordNotFound => exception
    fail(
      exception,
      "Account could not be found for token: #{account_token}"
    )
  end
{% endhighlight %}

Ahh, I'm starting to feel better now. No more digging, I can clearly see which
account token is missing and action can be taken right away to fix the problem.

Can we do better though? Like waking up for work and realizing it's a holiday good?
Can we make it even more obvious the problem our application is currently
encountering? In the words of an (in)famous Alaskan governor, "you betcha."

The first thing I'd like to do is create (override) a single method for fetching
accounts using a token:

{% highlight ruby %}
class Account < ActiveRecord::Base
  def self.find_by_token!(account_token)
    super
  rescue ActiveRecord::RecordNotFound => exception
    fail(
      exception,
      "Account could not be found for token: #{account_token}"
    )
  end
end
{% endhighlight %}

Okay, now we have a single way to find an `Account` by a token but I still want the
errors to jump out at me when I see them in my error service. They could still get
lost in other `ActiveRecord::RecordMissingError`s for something unrelated. Did
anyone order a custom exception?

{% highlight ruby %}
class Account < ActiveRecord::Base
  AccountTokenMissing = Class.new(StandardError)

  def self.find_by_token!(account_token)
    super
  rescue ActiveRecord::RecordNotFound
    fail(
      AccountTokenMissing,
      "Account could not be found for token: #{account_token}"
    )
  end
end
{% endhighlight %}

Yep, if I see that, I know exactly what is wrong. Even better, anyone else who
views that error knows we have a missing `Account` token. We've gone from an ambiguous
`ActiveRecord::RecordNotFound` exception to a specific one related to this scenario
and information that will help us fix the problem right away.

There's one other thing I'd like to see; the errors that are probably caused by
bad code vs a configuration error. Remember how we said tokens can just not be
sent in if someone broke something outside our system. I want to
highlight those errors so they can get extra attention.

{% highlight ruby %}
class Account < ActiveRecord::Base
  AccountTokenMissing = Class.new(StandardError)
  BlankAccountToken   = Class.new(StandardError)

  def self.find_by_token!(account_token)
    if account_token.blank?
      fail BlankAccountToken, 'Account token must be present'
    end

    super
  rescue ActiveRecord::RecordNotFound
    fail(
      AccountTokenMissing,
      "Account could not be found for token: #{account_token}"
    )
  end
end
{% endhighlight %}

I'll tell you what, that doesn't just feel good, it feels great. It may not look
like much on the surface but imagine the time saved for developers if they don't
have to research errors where we know exactly the problem. Now add that up over
all the places where problems arise of the same nature.

We've done good and helped our team become more efficient problem solvers.
