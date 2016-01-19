---
layout: post
title:  "Avoiding Fouls With Scheduled Sidekiq Jobs"
date:   2016-01-19 08:00:00 -0500
author: "Ryan Norman"
tags:   "ruby, rails, sidekiq, background jobs, asynchronous, developer, programming, coding"
---
Sidekiq: what a wonderful, frustrating tool. So much time spent diagnosing why some jobs fired and why others did not. The most headaches come from jobs that are scheduled to run at some point in the future. The reasons can range from the timing being off to a job executing that is no longer relevant. The last one has been especially difficult for our team to get right but through some trial and error we've found some best practices.

To illustrate the problems we've run into and how we solved them, I'll use the slightly contrived scenario of a basketball game's shot clock. For those not knowledgable about this rule in basketball, each team gets 30 seconds to take a shot when they take possession of the ball, otherwise a foul will be called.

We'll use a game and team objects to model this concept.

{% highlight ruby %}
class Game
  attr_accessor :id, :home_team, :away_team

  def get_team(team_name)
    home_team.name == team_name ? home_team : away_team
  end
end

class Team
  attr_accessor :name, :has_possession, :took_possession_at

  def set_possession
    self.has_possession     = true
    self.took_possession_at = Time.now
  end

  def remove_possession
    self.has_possession     = false
    self.took_possession_at = Time.now
  end
end
{% endhighlight %}

In order to have a foul called when a team is in violation of the shot clock, we'll use a worker that executes 30 seconds in the future as soon as possession is taken.

{% highlight ruby %}
class ShotClockFoulCaller
  include Sidekiq::Worker

  ShotClockViolation = Class.new(StandardError)

  def perform(game_id, team_name)
    if Game.find(game_id).get_team(team_name).has_possession
      fail ShotClockViolation, "#{team_name} failed to take a shot in time"
    end
  end
end
{% endhighlight %}

This job will be enqueued to perform for the length of the shot clock in the future:

{% highlight ruby %}
# game.rb
SHOT_CLOCK_LENGTH = 30.seconds

def give_possession(team_name)
  if home_team.name == team_name
    offensive_team = home_team
    defensive_team = away_team
  else
    offensive_team = away_team
    defensive_team = home_team    
  end

  offensive_team.set_possession
  defensive_team.remove_possession

  start_shot_clock(offensive_team)
end

def start_shot_clock(team)
  ShotClockFoulCaller.perform_at(SHOT_CLOCK_LENGTH.from_now, id, team.name)
end
{% endhighlight %}

This code will correctly raise an error if a team still has possession of the ball after 30 seconds. The problem arises if they retain possession after a missed shot or steal the ball back from the team before the job fires. It'll still call a foul even though the clock should've reset for them. There is an obvious, flawed way to stop this issue: just delete the job that no longer needs to execute.

{% highlight ruby %}
# game.rb
def start_shot_clock(team.name)
  Sidekiq::ScheduledSet.new.each do |job|
    job.delete if job.klass == 'ShotClockFoulCaller' && jobs.args.first == id
  end
  ShotClockFoulCaller.perform_at(SHOT_CLOCK_LENGTH.from_now, id, team.name)
end
{% endhighlight %}

Great, now we won't have fouls called on teams that really aren't in violation of the rule. What about the new problem we introduced? Oh, it's not obvious? Come on, you're better than that!

Imagine we have tens of thousands of games, all being played at the same time, all firing and deleting their own Sidekiq `ShotClockFoulCaller` jobs. How is your performance going to hold up having to step through all those jobs just to find the correct one to delete? It quickly becomes apparent that this solution is not going to scale well.

*So, what options do we have that'll stop the old job but won't bog our system down when our Ruby on Rails basketball simulator becomes more popular than Google?*

We have a couple different options and they all will result in passing in some more information to our Sidekiq worker that will help make decisions on whether or not to raise a shot clock violation. The easiest thing we can do is start tracking when a team took possession of a ball and passing this as an argument to the worker.

{% highlight ruby %}
# game.rb
def start_shot_clock(team.name)
  ShotClockFoulCaller.perform_at(
    SHOT_CLOCK_LENGTH.from_now,
    id,
    team.name,
    team.took_possession_at.to_s(:db), # Sidekiq will convert to string anyways
  )
end

# shot_clock_foul_caller.rb
def perform(game_id, team_name, possession_started_at)
  possessing_team = Game.find(game_id).get_team(team_name)

  if violated_shot_clock?(possessing_team, possession_started_at)
    fail ShotClockViolation, "#{team_name} failed to take a shot in time"
  end
end

def violated_shot_clock?(team, possession_started_at)
  team.has_possession &&
  team.took_possession_at.to_s(:db) == possession_started_at
end
{% endhighlight %}

[Boom goes the dynamite!](https://www.youtube.com/watch?v=m_djk1RQ2Ew) Well, actually quite the opposite; now we won't be blowing up errors for a team that has not committed a shot clock violation.

Passing in extra state-ful parameters can take other forms than just datetimes. A just as effective option would've been keeping track of how many possessions are in the game and only raising the exception if still on the same possession number. No matter what you choose, taking this extra forethought in passing in meaningful parameters can improve your Sidekiq jobs resiliency and predictability.
