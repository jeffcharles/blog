---
title: Lessons Learned from Developing a Django Commenting System
---
Over the last few months, I've been developing a new commenting system for [Open Source at Laurier](http://www.wluopensource.org/). While Django's commenting system does offer a lot, I felt that we needed more functionality.

Specifically, I wanted a commenting framework that would allow:

* replies to comments (limited to one level of depth)*
* editing of comments
* voting on comments
* a user being able to delete their own comments
* a comment form without name, email, or URL fields for authenticated users
* paginating comments
* banning users based on IP address
* sorting comments by newest, oldest, and top

Frankly the open source alternatives to Django's built-in comments framework are somewhat sparse outside of [django-threadedcomments](https://github.com/ericflo/django-threadedcomments) which only partially meets one of my requirements. I figured the only realistic option was to develop my own system on top of Django's. I decided to build on top of Django's after reading into how their commenting system is supposed to be extensible.

I thought adding in support for the requirements listed above would be trivial, but it certainly wasn't. I'll go over some of the lessons I have learned from trying to develop a Django-based commenting system:

* Usability and user experience are not correlated with flexibility
* Immediate feedback is important
* Modelling a threaded commenting system is not trivial
* Django's built-in commenting system has some &ldquo;special&rdquo; code

## Usability and user experience are not correlated with flexibility

Initially, I had planned to allow an unlimited number of levels of replies to comments (i.e., a user could reply to a reply to a comment). My thinking at the time was that more flexibility is automatically better for the end user. Why cap out the ability to reply to a comment and not allow a reply to a reply? If I have the technical ability to provide that functionality, then why should I limit what the end user can do by not including that functionality?

An important realization I had was that there were two types of users who would be using this system. There will be those who write comments but there are also users who would be content to simply read them. Well, what about the users who want to read comments? I believe a reader's mental context is tied to a thread of conversation. Assuming this is the case, arbitrarily branching threads breaks that context repeatedly. If you've ever tried having two separate conversations at once, you've probably found the experience unpleasant. I think reading through a multi-threaded commenting system is similar. Reading along a thread and then needing to go back to an earlier point in that thread to where another branch starts requires you to either mentally move backward or just dump the context of whatever you just read. This is similar to how you need to rewind or drop the context of one conversation to successfully participate in a different simultaneous conversation.

I think that example proves that flexibility is not always beneficial in regards to usability. There are situations in which a decision can diametrically affect two use cases. Sometimes you need to impose a limit on one use case so that a more common use case is made more pleasant.

## Immediate feedback is important

I recently heard a talk by [Alistair McKinnell](http://www.valuablecode.com/) about the developer testing landscape at a developer conference held by my employer, Desire2Learn. He started off this talk by explaining how he used to need to spend days going over a punch-card program to ensure there were no syntax errors before it would be shipped down to a mainframe to be run. There would be a three day wait to find out if there were indeed any errors.

As software developers, we do take it for granted that finding syntax errors is almost instant with modern technology and practices. However instead of waiting three days to find out if there are any syntax errors, in environments without unit tests, we wait a short period of time to find out if there were any obvious semantic errors in our code. In environments without continuous integration, we wait a longer period of time to see if all the components we've been developing work together. Finally, outside of truly agile projects, we wait an even longer period of time to find out if our software actually meets our users' needs.&nbsp;I think all software developers see the value in having immediate feedback for syntax errors. Unfortunately, investing in decreasing the feedback time for semantic errors and integration problems seems to be something that many developers need to be convinced to do.

In terms of using automated tests, I added some automated integration tests very late in the project, but still saw the value. I could feel confident that any change I made in my server-side code did not break any other parts of the code. It's a liberating feeling to know that you are free to refactor and make changes without needing to worry about breaking another part of your code. I regret not adding these tests sooner as it would have accelerated development.

Continuous integration is also a big deal. I did not do a good job of that at all in this project. I had worked on adding some authentication features through OpenID and Facebook Connect using the [django-socialregistration](https://github.com/flashingpumpkin/django-socialregistration) app. I had also started using the [django-debug-toolbar](https://github.com/robhudson/django-debug-toolbar). These two apps do not work at all well together. Trying to log in with OpenID while having the debug toolbar open results in a nasty 500 error. I still have not fixed this issue which makes testing OpenID authentication on my local developer instance painful. There also turned out to be an issue with our Facebook Connect and Twitter oAuth components not working at all on our production site. Continuous integration would have revealed these problems a lot sooner which would have made dealing with them easier.

## Modelling a threaded commenting system is not trivial

On the surface, modelling a threaded commenting system does not seem daunting. Simply take the Django comment object, create a one-to-one relationship with an OSL comment object, which then has a reference pointing at another Django comment object which points at another OSL comment object.

The three aspects that made things rather interesting were:

* There are alternative ways to store hierarchical data in a relational database
* Retrieving a list of comments correctly sorted with the correct attributes set is not something Django's ORM can do
* Django's ORM cannot delete an OSL comment object

There are four different ways that I've come across to store hierarchical data in a relational database:

1. The adjacency list (each record contains a foreign key pointing to its parent)
2. The nested set <http://dev.mysql.com/tech-resources/articles/hierarchical-data.html>
3. The materialized path (have a field with a list of delimited ancestor identifiers)
4. Ancestor tables (table contains three fields: ancestor id, node id, level)

Each required investigation to see if it would be an appropriate design. I chose to stick with an adjacency list because it resulted in less corner cases where bugs could crop up. I was also not certain how I would implement multiple sorting methods in the other three methods. The read performance isn't ideal but I think caching should help.

It took some effort to come up with the logic that would drive each sorting method. There's not an immediately apparent field to sort by. The end result is viewable at: <http://gitorious.org/osl-main-website/osl-main-website/blobs/master/wluopensource/osl_comments/models.py#line71>.

Finally, I'm not sure what it is about the model but I'm guessing it's something to do with the chain of foreign keys that are confusing the ORM when it tries to delete an OSL comment object. I guess it's just something to be aware of if you have any models that contain multiple foreign keys to the same table with some foreign keys nullable.

## Django's built-in commenting system has some &ldquo;special&rdquo; code

Django is a very strong framework and its developers have a lot to be proud of, but some of the code powering its commenting framework is in need of improvement. The main complaints I have surrounding functionality are:

* the inability to paginate comments
* the inability to ban comments from certain IP addresses

While I appreciate that this functionality is not difficult to add, I would think that pretty much every commenting system requires pagination. As well, every public facing commenting system could certainly benefit from the ability to bar comments from people intent on causing mischief. Adding in functionality to addressing these use cases would go a long way to providing a better out-of-the-box experience for Django users.

Another major issue I have is around how comments are modelled. The way comment result-sets are most commonly retrieved involves filtering content type, object primary key, site, is public, and is removed. The database only has indexes on the comment primary key, content type, site, and user. This hurts the performance of querying comments needlessly. It's possible to add the indexes yourself in your database and it's also possible to patch the models file to include the indexes on the fields. I think it would be better if this were fixed upstream.

Another oddity that I came across was that the comment details form has email address set as a required field while the comments model has a field for email address that accepts an empty string. This played havoc with pre-filling user data for authenticated users as Django's authentication system does not require users to set an email address on registration. While this isn't hard to work around, it doesn't provide a good developer experience when the required information for models and forms surrounding authentication and commenting are not consistent in the framework.

The final gripe I'm going to include in this entry revolves around some code I found in django.contrib.comments.views.utils.py, specifically the next_redirect method.

Here is the code in Django 1.2:

{% highlight python %}
def next_redirect(data, default, default_view, **get_kwargs):
"""
Handle the "where should I go next?" part of comment views.
The next value could be a kwarg to the function (``default``), or a
``?next=...`` GET arg, or the URL of a given view (``default_view``). See
the view modules for examples.
Returns an ``HttpResponseRedirect``.
"""
    next = data.get("next", default)
    if next is None:
        next = urlresolvers.reverse(default_view)
    if get_kwargs:
        joiner = ('?' in next) and '&amp;' or '?'
        next += joiner + urllib.urlencode(get_kwargs)
    return HttpResponseRedirect(next)
{% endhighlight %}

While the code above works in a large number of cases, there's one use case when it fails spectacularly. That case is when the data dictionary contains an entry for next that includes a URL fragment. For example, if you pass a value like `/articles/view/an-article/#c15`, you get a redirect to `/articles/view/an-article/#c15?c=15` which obviously does not work as intended. The main reason it came up with my commenting application was that I wanted to use a URL with a fragment to redirect users to the comment they had just posted or edited. I fixed that bug by wrapping the comment post view with my own view which manipulated the HttpResponse object if it was a redirect.

{% highlight python %}
response = comments.post_comment(request, next, using)
if response.status_code == 302:
    # Move the comment pk in the query string to the URL fragment
    redirect_location = response['location']
    redirect_url = list(urlparse.urlparse(redirect_location))
    redirect_qs = urlparse.parse_qs(redirect_url[4])
    comment_pk = ''
    if 'c' in redirect_qs:
        comment_pk = redirect_qs['c'][0]
        del redirect_qs['c']
    redirect_url[4] = urllib.urlencode(redirect_qs, True)
    redirect_url[5] = ''.join(['c', comment_pk])
    response['location'] = urlparse.urlunparse(redirect_url)
{% endhighlight %}

The `next_redirect` code should be using the `cgi`, `urlparse`, and `urllib` libraries to parse and manipulate the URL rather than using string manipulation. Library and utility authors should not be making assumptions about what data they are going to get and should be using the appropriate techniques to manipulate complex data like URLs.

In conclusion, I hope you've found this blog post informative. I'm very new to blogging so please feel free to leave some constructive feedback.
