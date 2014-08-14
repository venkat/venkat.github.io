---
title: Expanding Nested Relationships in your REST API
layout: post
category: programming
---

As you start working on the REST API for your awesome service, you will eventually encounter nested relationships and the need to figure out how you are going to deal with them.

For example, let us assume your REST API deals with two resources. A Comment resource (comments that users leave on your site) and the User resource.

A sample comment, in JSON:
{% highlight json %}
{
id: 234,
comment: "Sample comment",
postedAt: "2014-08-21T16:45:31.412Z"
}
{% endhighlight %}

<p></p>
A sample user, in JSON:
{% highlight json %}
{
id: 41,
name: "Venkat"
}
{% endhighlight %}
<p></p>
So far so good but notice that the comment is not connected to the user. You would like to know which user left the comment and display their name with the comment. This is where things get tricky and you have to figure out how to represent that relationship when the client fetches the comment(s) through your API.

You have two basic options to begin with - option one, have a URL or id reference to the user in the comment, like so:
{% highlight json %}
{
id: 234,
comment: "Sample comment",
postedAt: "2014-08-21T16:45:31.412Z"
user: /users/41/
}
{%endhighlight%}

<p></p>
This way, you can now do a HTTP GET on /users/41/ when you want to get the details of the user who left the comment with id 234. This approach has three main advantages - you limit the amount of data you transfer, you just need to send the reference id or URL of the user, not the entire resource. Second, it is efficient, you save the time that would have been spent on serializing the user resource. Third, cuts off a database lookup. This is possibly the biggest win - as my guru Tom Christie notes, lookups can [significantly slow down your service](http://www.dabapps.com/blog/api-performance-profiling-django-rest-framework/).

But there is one significant disadvantage, network latency. You can only fetch the user resource (over the network) once you fetch the comment and get the user reference from it. So, if you need to list 10 comments, you will have to make a bunch of HTTP GETs to get the details of the users who left those comments. This is not usually a bad thing (specially if you have cache-control setup right) if your design  and UI allow you to have some empty placeholder and the details can be loaded up asynchronously as each GET succeeds. In our mobile world, you should factor in the user moving in and out of data connectivity and as a result (based on the order in which the HTTP operations are executed), the more the number of pending HTTP GETs, the higher the chances of some of them failing and so you should be OK with not being able to display some of the user details. 

In certain cases, this is not acceptable - let us say you're showing a complex receipt to your user which has a bunch of inter-related details. For example, the details of the charge, the details of the service provider, credits applied etc., In such cases, it is preferable to be as complete as possible and show all details together. Doing this by fetching disparate inter-connected information over multiple HTTPS GETs and have it be all-or-nothing can be a headache. You can still do it by showing some kind of loading spinner as you keep track of the GETs that are in progress and the ones that are completed in the background and that brings us to option two - you can just "expand" the nested relationships, where you roll-out the related resources in place like so:

{%highlight json %}
{
id: 234,
comment: "Sample comment",
postedAt: "2014-08-21T16:45:31.412Z"
user: {
        id: 41,
        name: "Venkat",
      }
}
{%endhighlight%}

<p></p>
This approach has the flip of the pros and cons we saw with the previous approach. You do less network calls but end-up with more database lookups, more serialization and more data transferred everytime, even if your client ends up not  needing most of the nested information in most cases. This is specially pronounced when you are listing N objects. If you do M lookups per object then you need to do N x M lookups to list them all.

Usually, REST API authors choose one approach or the other at the API level or the endpoint level. They might even have two versions of the API, with and without the expansion.

There are limitations to expanding all relationships in every resource. If a resource is connected to a bunch of other resources, the number of database lookups needed to respresent a single instance of the resource can be prohibitively high. Also, how do you choose what level of depth you expand the relationships for?

There is a third, better approach that I learned from the famous [Stripe API](https://stripe.com/docs/api) - optionally specifying which relationships to [expand](https://stripe.com/docs/api#expand). You do this with the help of a special query parameter, say `expand`. The client chooses which attributes of the resource it is fetching by specifying them as the value of the `expand` key when it does a HTTP GET. Multiple attributes that need expanding are seperated by comma and nested attributes are represented by a dot. Using query paramters to support extra features on your REST API is not a new idea. Most REST APIs have some sort of sorting or filtering or pagination exposed through this approach. This is an extension of that idea to support selective expansion. JIRA's REST API is another [example](https://jira.atlassian.com/rest/api/latest/issue/JRA-9?expand=names,renderedFields) which supports `expand`,  where certain attributes are completely hidden unless specified with the expand query parameter. The resulting JSON has a special attribute which lists the names of such hidden, expandable attributes. Netflix's REST API also supports expansion [in a very specific way](http://developer.netflix.com/docs/REST_API_Conventions).

This level of flexibility works great for your API users. They know best about their application constraints and they can choose to only expand the specific resources they need right away and doing asynchronus GETs for others based on the experience they want to give their customers. This gives them the level of ease and control they would appreciate and as a result, the server only needs to do just the exact amount of work needed.

If you use the Django REST Framework, here is an example of how you could support expansion in your API - [https://github.com/venkat/DRFInlineExpansion](https://github.com/venkat/DRFInlineExpansion).

Let see how it looks if we use inline expansion on the Snippet resource mentioned in the Django REST Framework [tutorial](http://www.django-rest-framework.org/tutorial/1-serialization#creating-a-model-to-work-with).

HTTP GET /snippets/

{% highlight json %}
{
    "count": 1, 
    "next": null, 
    "previous": null, 
    "results": [
        {
            "url": "http://localhost:8000/snippets/1/", 
            "highlight": "htep://localhost:8000/snippets/1/highlight/", 
            "title": "test", 
            "code": "def test():\r\n     pass", 
            "linenos": false, 
            "language": "Clipper", 
            "style": "autumn", 
            "owner": "http://localhost:8000/users/2/", 
            "extra": "http://localhost:8000/snippetextras/1/"
        }
    ]
}
{% endhighlight %}

Now, here is the JSON if you want the same listing but with the owner resource expanded inline:

HTTP GET /snippets/?expand=owner

{% highlight json %}
{
    "count": 1, 
    "next": null, 
    "previous": null, 
    "results": [
        {
            "url": "http://localhost:8000/snippets/1/", 
            "highlight": "http://localhost:8000/snippets/1/highlight/", 
            "title": "test", 
            "code": "def test():\r\n     pass", 
            "linenos": false, 
            "language": "Clipper", 
            "style": "autumn", 
            "owner": {
                "url": "http://localhost:8000/users/2/", 
                "username": "test", 
                "email": "test@test.com"
            }, 
            "extra": "http://localhost:8000/snippetextras/1/"
        }
    ]
}
{% endhighlight %}

And that's it! Go forth and add expansion support to all your APIs!
