---
layout: post
title:  "Modeling non-CRUD Operations in RESTful APIs"
date:   2020-05-20 18:38:29 +0300
categories: rest
---
Are you a purist when it comes to designing RESTful APIs? Well, I kind of am.
However, our software engineering life dictates its own rules, and we often just have to be pragmatic.

Tons have been written about REST API design, the best practices, and the
[Richardson maturity model](https://martinfowler.com/articles/richardsonMaturityModel.html).
Surprisingly, I could find very little on modeling _non-CRUD_ operations in REST.
Yes, that is right, sometimes there is a need to express workflows beyond creating,
reading, updating, or deleting a resource. Or it is much simpler to do so.

I was no less surprised that the worst (in my opinion) option also seemed to be the most popular.
Let me show you better ones.

# 1. Generic Actions Endpoint

Let us start with the option I hate, yet keep coming across time and again.
A request is sent to the `/actions` or `/requests` endpoint of a resource,
with action-specific attributes for each action. For example

```
POST http://api.examle.com/jobs/321/actions

{
    "action": "start",
    "delay": "30s"
}
```
and

```
POST http://api.examle.com/jobs/321/actions

{
    "action": "stop",
    "reason": "Duplicate"

}
```

Well, an endpoint that is used to send multiple types of actions just _is not explicit enough_.
It is hard to figure out what actions are available. Moreover, action parameters often vary,
making it impossible to have a common schema for the input, and reducing the clarity even further.

If understanding the contract of such an API can be tough, documenting it is plain painful.
Imagine trying to describe all the possible combinations of actions, their parameters,
and output (structure and error codes) in [Swagger](https://swagger.io/).
And good luck with rendering the documentation page!

Finally, a generic endpoint is a blessing for sloppy developers.
Need to do something with a resource? Just throw in another action!

```
POST http://api.examle.com/jobs/321/actions

{
    "action": "add-owner",
    "user": "jane.doe@example.com"

}
```

But wait, are we building REST or RPC?

# 2. Endpoint per Action

On the other hand, state transitions via a dedicated "sub-resource"
are both explicit and allow sending action-specific complex input:

```
POST http://api.examle.com/jobs/321/start

{
    "reason": "Customer request",
    "effective": "2019-05-11"

}
```

A single, action-specific endpoint is also much easier to expand
(do not forget backward compatibility!) and document.

Some of the popular APIs, such as [Stripe](https://stripe.com/docs/api) and
[Blogger](https://developers.google.com/blogger/docs/3.0/reference/), use this approach.

# 3. State Attribute

Another approach is to update a `state` or `status` field with a desired end-state value.

This may work in some cases, but it may also be confusing if changing the state takes some time
and is reported not by a user, but by some background process on the server
(i.e. there is a _desired_ state by the user and an _actual_ state by the server).
Consider, for example, sending an email. Should the user update the state to `"sent"` or `"sending"`?
In the former case, the state will not be accurate. In the latter, it will be written to by two actors,
each of them allowed to use only some values but not the others.

If an action has side-effects and is not idempotent, updating an attribute will violate the conventions,
because the only verb for non-idempotent updates is `POST`. And you do not `POST` a field.
Let us assume that we had an email with `{ "status": "draft" }`.
Updating it to `{ "status": "sent" }` as follows is easy:

```
PUT http://api.examle.com/emails/xyz

{
    "status": "sent",
    "to": "jane.doe@example.com",
    "subject": "Dinner",
    ...
}
```

or

```
PATCH http://api.examle.com/emails/xyz

{
    "status": "sent"
}
```

But what should happen when we call the API again? Do we expect the email to be resent?
Or should an attempt to resend an already sent email be ignored &mdash; made idempotent?

Keep in mind also, that some states may be orthogonal, for example, `archived` and `complete`.
That is, an item may be _archived_ regardless of whether it is _complete_ or _incomplete_.

_Reflecting_ the state of a resource in a field, on the other hand, is fine.
It has an additional benefit of including useful information about the state or allowing to
communicate valid state transitions. See, for example, how it can be combined with the
[Endpoint per Action](#2-endpoint-per-action) approach:

```
GET http://api.examle.com/jobs/321

{
    "state": "running",
    "next": [
        {
            "name": "stopped",
            "href": "https://api.example.com/jobs/321/stop"
        }
    ]
}
```

Or better yet

```
GET http://api.examle.com/jobs/321

{
    "state": {
        "name": "running",
        "since": "2019-05-18T13:28:44+00:00"
    },
    "links": [
        {
            "rel": "stop",
            "href": "https://api.example.com/jobs/321/stop"
        }
    ]
}
```

Or even simpler

```
GET http://api.examle.com/jobs/321

{
    "state": {
        "name": "running",
        "since": "2019-05-18T13:28:44+00:00"
    },
    "self": "https://api.example.com/jobs/321",
    "stop": "https://api.example.com/jobs/321/stop"
}
```

Notice that `state` in the last two examples is a complex type.
I recommend structuring it this way because otherwise trying to add information about the state
will break the backward compatibility of the API. I am talking about going from
`"state": string` to `"state": { object }`.

The links can help figure out navigation where the endpoints do not exactly follow the RESTful convention.

You can use two separate states, of course &mdash; one desired and the other actual. I think it is even preferable.

```
GET http://api.examle.com/jobs/321

{
    "desiredState": {
        "name": "stopped",
        "requested": "2019-05-18T13:28:44+00:00"
    },
     "actualState": {
        "name": "running",
        "since": "2019-05-01T10:21:18+00:00"
    }
}
```

# 4. Collection of Sub-Resources

A repeatable action, though, can be modeled pretty nicely with a noun (REST!),
where `/thing/{thingId}/action-requests` describes a ledger of all requests of the same type,
and POSTing to it creates a new such request.
That is, `/message/{id}/advertise` becomes `/message/{id}/advertisements/`.

Let us see how this is supposed to work.

```
POST http://api.examle.com/message/321/advertisements/

{
    "target": "Facebook"
}
```

Another POST:

```
POST http://api.examle.com/message/321/advertisements/

{
    "target": "AdSense"
}
```

And now a GET:

```
GET http://api.examle.com/message/321/advertisements/

[
    {
        "target": "Facebook",
        "time": "2019-05-18T13:28:44+00:00",
        "status": "Success"
    },
    {
        "target": "AdSense",
        "time": "2019-06-11T11:55:03+00:00",
        "status": "Failed"
    }
]
```

# Conclusion

In this article, we have seen four different ways to model actions in REST,
each with its pros and cons. Hopefully, you have a bit more useful tools under your belt now.

Finally, I would like to remind you that REST is not a solution to every problem.
If you are trying to model a workflow, or have many non-CRUD operations in your API,
you should probably consider a different API style.