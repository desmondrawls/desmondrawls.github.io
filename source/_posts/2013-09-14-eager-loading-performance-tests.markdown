---
layout: post
title: "Eager Loading Performance Tests"
date: 2013-09-14 22:09
comments: true
categories: 
---

Recording with multiplexing inspected by firebug from trans-atlantic fiber-optic cables aka the 4ms arbitrage industry aka America the startup aka the fast-fingers words-per-minute hall-of-fame aka the prepared-exclusively-for-Avi-Flombaum library aka the mayor's school-for-kids-that-can't-code-good-and-want-to-learn-to-do-other-things-good-too aka the Flatiron School in exile

Eager loading is boss. But exactly how boss is it? I wanted to measure how helpful the :includes method is so I spent my Saturday night rigging up some performance tests. 

I determined that the advantage you get from eager loading depends on how many records you are fetching as much as it depends on how many associated tables you are drawing those records from.

In order to test references to associated tables I am using methods that call each of the associations individually. For example, for one reference I have:
{% codeblock %}
  #/app/models/project.rb
  def self.creators
    projects = self.all
    projects.each do |project|
      project.creator.name if project.creator
    end
  end
  
  def self.creators_with_includes
    projects = self.includes(:creator).all
    projects.each do |project|
      project.creator.name if project.creator
    end
  end
{% endcodeblock %} 

And for seven references I have:
{% codeblock %}
  def self.creators_users_ideas_teams_directors_competitors_and_industries
    projects = self.all
    projects.each do |project|
      project.creator.name if project.creator
      project.user.name if project.user
      project.idea.name if project.idea
      project.team.name if project.team
      project.director.name if project.director
      project.competitor.name if project.competitor
      project.industry.name if project.industry
    end
  end

  def self.creators_users_ideas_teams_directors_competitors_and_industries_with_includes
    projects = self.includes(:creator, :idea, :team, :director, :competitor, :industry).all
    projects.each do |project|
      project.creator.name if project.creator
      project.user.name if project.user
      project.idea.name if project.idea
      project.team.name if project.team
      project.director.name if project.director
      project.competitor.name if project.competitor
      project.industry.name if project.industry
    end
  end
{% endcodeblock %}   


Here is the graph of my experiment varying the number of records being retrieved from the database (the red line is with eager loading):

<img src="/images/eager-records.jpg" alt="tables" height="300" width="1000" display="inline"> 

You save more time when you are retrieving more records. You save approximately half a millisecond per record.

Here is the graph of my experiment with varying the number of tables being referenced (again, the red line is with eager loading):

<img src="/images/eager-associations.jpg" alt="tables" height="300" width="1000" display="inline"> 

This graph looks very similar to the previous one where I vary the number of records. The similarity makes sense. The way I have my methods set up I am retrieving 100 records from each additional table. So it's not surprising that the performance advantage that comes with eager loading one more table is approximately the same as the performance advantage that comes with eager loading 100 more records. 

Without eager loading every record comes from a separate database query that doesn't care whether it is returning to the same table over and over or hitting a different table every time. With eager loading you have one initial database query then, after that, every relational call is referencing our stored array of hashes.

So the answer is eager loading saves you approximately half a millisecond per record. 