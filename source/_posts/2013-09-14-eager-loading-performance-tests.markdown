---
layout: post
title: "Eager Loading Performance Tests"
date: 2013-09-14 22:09
comments: true
categories: 
---

Recording with multiplexing inspected by firebug from trans-atlantic fiber-optic cables aka the 4ms arbitrage industry aka America the startup aka the words-per-minute hall-of-fame aka the prepared-exclusively-for-Avi-Flombaum library the mayor's school-for-kids-that-can't-code-good-and-want-to-learn-to-do-other-things-good-too aka the Flatiron School in exile

Eager loading is boss. I wanted to know how exactly how helpful the :includes method is so I spent my Saturday night rigging up some performance tests. 

I determined that the advantage you get from eager loading depends on how many records you are fetching from the database. I was surprised to find that it makes no difference how many associated tables you are referencing. I had, previously, assumed there would be a linear relationship between the number of tables and the time saved. If you save 100ms eager loading one associated table, shouldn't you save 200ms eager loading two tables? Somehow, that's not the case. The associations don't add up. Rather, eager loading saves time per record. It depends on how tall the tables you're referencing are, not how many tables you're referencing.

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
    projects = self.includes(:creator).all
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


Here is the graph of my experiment with varying the number of records being retrieved from the database:

<img src="/images/eager-records.jpg" alt="tables" height="300" width="1000" display="inline"> 

You save more time when you are retrieving more records.

Here is the graph of my experiment with varying the number of tables being referenced.

<img src="/images/eager-associations.jpg" alt="tables" height="300" width="1000" display="inline"> 

The time saved referencing a single associated table is the same as the time saved referencing seven associated tables. How is this possible? If I have 100 project records and I reference seven associated tables I am pulling up 700 more records in 7 different tables. I would expect this to be significantly slower than pulling up 100 more records in 1 table.

If you can explain this phenomenon please e-mail me at captaingrover@gmail.com