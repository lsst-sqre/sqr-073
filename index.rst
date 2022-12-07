:tocdepth: 1

Abstract
========

The Rubin Science Platform is a shared computing environment and thus is vulnerable to resource starvation or excessive cost if one user (possibly accidentally) consumes too many resources.
We therefore want to impose automatically-enforced limits on the resources available to a user.
In some cases, these can be statically enforced or delegated to underlying infrastructure, such as resource limits for user notebooks.
In other cases, such as API rate limits, they need to be dynamically calculated and imposed by Rubin code.

This tech note discusses the requirements and possible implementation options and proposes an initial design.

.. note::

   This is part of a tech note series on identity management for the Rubin Science Platform.
   The primary documents are :dmtn:`234`, which describes the high-level design; :dmtn:`224`, which describes the implementation; and :sqr:`069`, which provides a history and analysis of the decisions underlying the design and implementation.
   See the `references section of DMTN-224 <https://dmtn-224.lsst.io/#references>`__ for a complete list of related documents.

Requirements
============

We have the following overall goals for quotas:

#. Limit scaling costs for the Science Platform as a whole.
   Deployments may have a fixed pool of resources that cannot be easily increased and need to be shared by all users.
   Even when scaling is possible, such as at the Cloud Data Facility, we want to cap the amount of project money spent on resources.

#. Limit users to a "fair" share of resources on a deployment of the Science Platform.
   Fair in this context primarily means starvation-avoidance: overuse of a service by one user should not make that service unavailable or unusable by other users.

#. Support allocation of additional resources to specific groups of users.
   Based on funding, collaborations, in-kind contributions, and other factors, some deployments of the Science Platform may want to change resource limits for certain groups of users.

#. Emergency throttling of Science Platform services under heavy load.
   We want to have a mechanism where we can shed load and start rejecting requests, hopefully in a fair way that avoids starvation, if the Science Platform as a whole or individual services are overloaded and unable to keep up with incoming requests.

Those translate into the following operational requirements:

#. Users should be able to see their quotas and, where appropriate, their current usage so that they know if they're approaching their quota.

#. The quota system must support default quotas for all users, additional quota increments for specific groups, and emergency overrides (in the event that the platform becomes overloaded) to temporarily reduce all quotas.

#. Emergency quota overrides should be done via an API and not require restarting services, since the cluster may be in a poor state when we're trying to put emergency quota rules in place.

Quota types
===========

In the initial implementation, we expect to have the following quota types:

- Total CPU and memory available in the Notebook Aspect, including any additional pods created via services like Dask_.
- API rate limits for all API services.
  In some services, we will want a separate quota for expensive requests (async TAP jobs, for example) from cheap requests (informational queries about the status of a job).

.. _Dask: https://www.dask.org/

We also want an emergency way to shut off creation of new user notebook pods regardless of whether a given user has access to the Notebook Aspect.
We may want to only block creation of new notebook pods, or block all access to notebook pods.
We may want to do this only for specific groups, or for everyone using the Science Platform.

Additional quota types we may want to add in the future:

- Disk quotas: user home directories and, separately, group-owned project directories.
  We currently don't have a mechanism for handling group-owned directories.

- Quotas on the size of user database tables.

- TAP query quotas based on the effort required to process the quota (as opposed to API limits on the TAP service itself).

This general framework is intended to support tracking other types of quotas, but only the first two types of quotas and the emergency API are discussed below.

Overall design
==============

Gafaelfawr_ will be responsible for tracking quota information.
Quota details for a user will be included in the response to ``/auth/api/v1/user-info`` for a user's token.

.. _Gafaelfawr: https://gafaelfawr.lsst.io/

Quotas are by-user, and all requests or resources consumed by all tokens for that user, including notebook and internal tokens, will count against the same quota.

Basic API rate limiting will be done by Gafaelfawr.
The auth URL configured for an ingress (usually generated via a ``GafaelfawrIngress`` custom resource) will configure the name of the service for quota purposes.
Gafaelfawr will then track the number of API requests for that service within a time interval.
If a user exceeds that limit, it will start rejecting requests with a 429 status code.

This is not as good as the more fine-grained understanding a service has about its requests, which would allow expensive requests to count against a quota but cheap requests to be free.
However, it allows easy deployment of basic quota support to every service.
More sophisticated quotas can be added later if needed.

Services that need to apply quotas directly can request a delegated token (possibly with no scopes if they don't need a delegated token for anything else) and then use that token to retrieve quota information via ``/auth/api/v1/user-info``.

Quota settings
==============

For the initial implementation, we will add quota information to the Gafaelfawr configuration via the Helm chart.

Quota configuration
-------------------

The data structure for the quota information will look something like this:

.. code-block:: yaml

   quotas:
     default:
       notebook:
         cpu: 9
         memory: "27Gi"
       api:
         datalinker: 500
         hips: 2000
         tap: 500
         vo-cutouts: 100
     groups:
       g_developers:
         api:
           datalinker: 500

The ``default`` key establishes default quotas for every user.
The ``groups`` key provides additional quotas to particular groups.
These quotas are additive, so in the above case a user who is a member of the ``g_developers`` group would have a quota of 1000 for the ``datalinker`` service.

API quotas are in requests per fifteen minutes.
This is an awkward interval, but (as discussed in :ref:`rate-limiting`) the interval is also the length of time that the user will be blocked from accessing the service.
One minute seems to short, and one hour (used by GitHub) seems too long.

The keys under ``api`` are the names of the services, as configured in the Gafaelfawr auth URL for that service.
Normally, this is set in the ``config`` section of the corresponding ``GafaelfawrIngress`` custom resource.

A given API service does not have to have a quota.
If no quota is configured, the quota for all users is unlimited and requests won't be tracked.

Quota overrides
---------------

Emergency override information will be stored in the Gafaelfawr Redis under the key ``quota-override``.
The value of the key will be a JSON document such as the following:

.. code-block:: json

   {
       "default": {
           "notebook": {
               "spawn": false,
               "cpu": 4
           },
           "api": {
               "datalinker": 10
           }
       },
       "groups": {
           "g_users": {
               "api": {
                   "vo-cutouts": 10
               }
           }
       },
       "bypass": [
           "g_admins"
       ]
   }

This mostly has the same structure as the configuration, but it overrides all quota information taken from the configuration, including additions from groups.
So, for example, if the above override were in place, all users would have a quota of 10 for the datalinker API, including members of ``g_developers`` who normally get an extra 500.
Members of the ``g_users`` group would only have a quota of 10 for the vo-cutouts API.

There is an additional key here under notebook, ``spawn``, which is a boolean that controls whether affected users are allowed to spawn new notebooks at all.
This allows quickly cutting off access to starting new Notebook Aspect pods for every user or only for users in particular groups, without changing token scopes.

Finally, the ``bypass`` key in the ``quota-override`` data lists groups excluded from the override.
In this example, members of ``g_admins`` can use the services according to the normal quota settings, without any changes from the override.

In the initial implementation, Gafaelfawr won't cache the quota override information and will try to retrieve it from Redis for every request potentially affected by quotas.
We'll see if that creates a performance problem and add in-memory caching if it does.

API
---

There will be three new Gafaelfawr APIs to get and set the quota overrides:

``GET /auth/api/v1/quota-overrides``
    Retrieves the current quota overrides in the above JSON format.
    Returns 404 if there are no quota overrides.

``PUT /auth/api/v1/quota-overrides``
    Creates or replaces the quota overrides.
    The body should be the above JSON format.
    There is no ``PATCH`` API; the complete override configuration has to be provided.
    (We don't expect to need much complexity or to use this that frequently.)

``DELETE /auth/api/v1/quota-overrides``
    Delete the quota overrides.
    Returns 404 if there are no quota overrides and 204 on success.

Options considered
------------------

The original plan had been to store quota information in the database and provide an API and eventually a UI for updating it.
However, that adds a lot of complexity and additional design for not a lot of expected benefit, at least for now.
Storing it in the configuration makes it harder to update quickly and makes updates more intrusive, but it's not obvious how frequently we will be updating quota grants.
Using configuration is much simpler to implement, and we can always switch to a database later.

The simplest approach would be to make everything configuration, but then, during an emergency, we would have to change the Gafaelfawr configuration and restart Gafaelfawr, which may be dangerous or undesirable under heavy load.
Being able to selectively override the normal configuration in Redis allows us to provide an API to change this on the fly, requiring only that Gafaelfawr be responsive.

Redis was chosen over the database as the place to store quota overrides, since Redis is much faster to query.

The rate-limit configuration for APIs is unsatisfying in both syntax and in semantics.
For syntax, ideally it would be specified as ``<count>/<time>`` so that both the number of requests and the time interval could be given.
But this makes the logic of adding in group quotas more complicated and confusing since they may use different time intervals.

For semantics, ideally we should only count "expensive" API calls, such as requesting a cutout or performing a TAP query, and not count "cheap" API calls, such as asking for the status of a job.
This in theory could be done via complicated rules in the ingress specifying how to match the URL and verb patterns of complex queries, but in practice that seems hard to maintain.
Alternately, we could assume that all ``GET`` requests are cheap and all requests with other verbs are expensive, but unfortunately IVOA standards require some expensive queries be accesible via ``GET``.

The current approach is the simplest and provides a general facility to impose basic rate-limits on anything, so we're going to start with it and see if it's adequate in practice.
If not, we may need to move more quota checking into the separate services rather than in Gafaelfawr.

Quota checking
==============

API
---

The ``/auth/api/v1/user-info`` route will be extended to add quota information.
The response will look like this:

.. code-block:: json

  {
      "username": "someuser",
      "name": "Alice Example",
      "email": "alice@example.com",
      "uid": 4123,
      "gid": 4123,
      "groups": [
          {
              "name": "g_special_users",
              "id": 123181
          }
      ],
      "quota": {
          "api": {
              "datalinker": 500,
              "hips": 2000,
              "tap": 500,
              "vo-cutouts": 100
          },
          "notebook": {
              "cpu": 9,
              "memory": "27Gi"
          }
      }
  }

The quota shown will be the calculated amount reflecting any additions from groups and any configured overrides.
The sources of the quota components will not be shown.
(We may eventually want to add a separate API to see the full quota breakdown of why a user has the quota that they do, but it's not part of the initial design.)

Notebook Aspect
---------------

The Notebook Aspect lab controller (see :sqr:`066`) will use its delegated notebook token during menu creation and lab creation to retrieve the user's quota information.
For the menu response, it will filter out any notebook sizes that exceed the user's quota.
For the lab creation, it will add a Kubernetes ``ResourceQuota`` resource for the user's namespace that sets limits matching the user's quota.

.. _rate-limiting:

Rate limiting
-------------

Currently, a ``GafaelfawrIngress`` only configures the name of the protected service when it is requesting a delegated token (as ``config.delegate.internal.service``).
This configuration will be moved up to ``config.service`` and correspond to a new ``service`` parameter to the ``/auth`` route, replacing ``delegate_to``.
Delegation will then be controlled by ``delegate_scopes``.

Rate limiting will then be done if and only if there is an API quota for a service whose name matches the ``service`` parameter.

To support multiple Gafaelfawr instances without confusing impact on quotas from having quotas be tracked separately by different instances, the data for quota enforcement will be stored in Redis.
Gafaelfawr's current Redis is used to store tokens, which are valuable data that needs to be persisted to disk and backed up, and for which writes are relatively rare.
The quota tracking data will require huge numbers of writes but is not valuable and does not need to be persisted.
We will therefore stand up a second Redis instance for quota tracking that is in-memory only with no persistent storage.

The rate limiting will be done using limits_.

.. _limits: https://limits.readthedocs.io/en/stable/index.html

The rate limiting algorithm is fixed window.
This means that the user will be allowed their quota of requests within a window of time (15 minutes).
At the end of that window, their quota will reset and they'll get their full quota of requests again.
There are more complex algorithms that are better at smoothing out load (sliding window, for instance), but fixed window is easy to explain and reason about, is extremely fast and cheap to represent in Redis, and matches the way GitHub does rate limiting.

If the user exceeds their rate limit, Gafaelfawr will reject all requests to that API with 429 error responses until the reset interval has passed.
The 429 response will include a ``Retry-After`` header (see `Retry-After_`).
This will require understanding how to configure NGINX to pass the actual reply from the auth request subhandler back to the client, rather than turning all unexpected errors into 500 errors.
Doing that work will also fix several other long-standing problems with Gafaelfawr.

.. _Retry-After: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Retry-After

Successful responses should also include ``X-RateLimit-Limit``, ``X-RateLimit-Remaining``, and ``X-RateLimit-Reset`` headers.
These have the same meanings as the headers without the leading ``X-`` specified in the Internet-Draft `draft-ietf-httpapi-ratelimit-headers <https://datatracker.ietf.org/doc/html/draft-ietf-httpapi-ratelimit-headers>`__.
This will require lifting headers from the auth subrequest response into the main response, which will require some NGINX work.

Options considered
------------------

Quota information could be included in structured form in an HTTP request header rather than requiring an API call, but we've moved away from that pattern elsewhere since the API call pattern is simpler and more straightforward.
The primary advantage of HTTP headers is to optimize away the API call to Gafaelfawr and the extra overhead in creating a delegated token, but we are avoiding premature optimization until we have evidence it is a problem.

There are many ratelimiting packages available in Python.
We chose limits_ because it supports Redis, asyncio, and the type of configuration that's required for use inside Gafaelfawr.
It also supports a wide variety of rate limit algorithms if we want to change fixed-window to something more sophisticated.
It unfortunately depends on a different Redis library (maintained by the same author), so this introduces a second Redis library into our infrastructure, but the other advantages outweighed this.

Other options considered:

- `fastapi-limiter <https://github.com/long2ice/fastapi-limiter>`__ wants to be invoked as a FastAPI dependency.
  This is great for rate limiting within a FastAPI application, and we should consider it again when we need to move rate limiting into the individual service, but it wants to run as a dependency and relies on being able to extract the route from the request.
  All Gafaelfawr rate limit checking happens in the ``/auth`` route, and Gafaelfawr needs to be able to rate limit on the basis of the user and service extracted from the token.

- `ASGI RateLimit <https://github.com/abersheeran/asgi-ratelimit>`__ has a similar problem: it wants to get all the configuration for the rate limiting and applies it by analyzing the incoming route.

- `aiolimiter <https://aiolimiter.readthedocs.io/en/latest/>`__ and `SlowApi <https://slowapi.readthedocs.io/en/latest/>`__ only work in-memory in a single process and don't support a shared rate limit in Redis.

- `python-redis-rate-limit <https://github.com/EvoluxBR/python-redis-rate-limit>`__ has a good API and the most sophisticated counter implementation using the Lua script recommended by the `Redis documentation <https://redis.io/commands/incr/#pattern-rate-limiter-2>`__.
  Unfortunately, it doesn't support asyncio, which is a requirement for Gafaelfawr.
  (It also has a 0.0.8 version number.)

- `asyncio-redis-rate-limit <https://github.com/wemake-services/asyncio-redis-rate-limit>`__ has all the required features, but the key generation algorithm seems dodgy to me and it uses a relatively unsophisticated fixed-window algorithm.

Finally, NGINX can do rate limiting directly.
This can be configured per-ingress with `annotations <https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#rate-limiting>`__.
Using this rate limiting would be the least effort for us.

However, since NGINX has no access to the user's authentication information, it cannot do rate-limiting by user, only by IP address.
Since we expect many requests to come from inside the cluster via other services such as the Portal or Notebook Aspect, this cannot be used for the type of rate limiting we want to do.
We may use NGINX rate limiting for :ref:`dos-protection`.

There are two basic ways to respond to a user hitting a rate limit: delay the request until the rate limit allows it, or reject the request.
We've chosen to reject the request, since delaying it requires queuing it on the server, which adds load to a potentially already-overloaded server and also may create complex timeout issues that are hard to debug.

The drawback of rejecting the request is that it may produce failures in long-running processes when one of their underlying requests is rate-limited.
This will place the onus on the user to retry rate-limited requests if needed.
We may want to add support to PyVO_ for retrying rate-limited requests automatically.

.. _PyVO: https://pyvo.readthedocs.io/en/latest/

Metrics
=======

Rate limit information will be logged as part of the log message for each authentication request to Gafaelfawr.

We will eventually want more data than that, particularly for rate limiting.
Ideally, Gafaelfawr should log metrics for how many users are being rate-limited, how many requests were rejected due to rate-limiting (and from how many distinct users), and how many users have reached 50% or 75% of a rate limit.
We don't yet have a general metrics framework for Gafaelfawr; once one exists, metrics like that will be added.

.. _dos-protection:

Denial of service protection
============================

This rate limiting system is intended to fairly share resources among non-malicious users issuing a normal rate of API requests.
Each request, even if rate-limited, requires processing by NGINX, an auth subrequest to Gafaelfawr, and processing by Gafaelfawr, including at least two Redis reads, one write, and often an LDAP lookup.
This means NGINX and Gafaelfawr could still be overloaded by higher quantities of traffic, such as runaway processes in tight loops or an intentional denial of service attack.

Fully defending against denial of service attacks is outside the scope of the Rubin Science Platform and not something we can reasonably expect to do.
But we can apply sanity limits on requests at the NGINX level to protect against being overwhelmed by accidents external to the cluster.

This can be done with `ingress-nginx annotations <https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#rate-limiting>`__, normally managed via ``GafaelfawrIngress``.
This rate limiting can only be done by IP address, not by user.
The NGINX rate limit should be higher than the quota of any given user, since it will be applied to every user and may apply to multiple users at the same time if they share an outbound IP address.

These rate limits must either be set high enough to allow for expected levels of traffic from in-cluster services that are making requests on the user's behalf, such as the Portal Aspect, or in-cluster services should be excluded from the rate limiting using ``nginx.ingress.kubernetes.io/limit-whitelist``.
