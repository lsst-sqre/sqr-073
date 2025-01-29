############################
RSP quotas and rate limiting
############################

.. abstract::

   The Rubin Science Platform is a shared computing environment and thus is vulnerable to resource starvation or excessive cost if one user (possibly accidentally) consumes too many resources.
   We therefore want to impose automatically-enforced limits on the resources available to a user.
   In some cases, these can be statically enforced or delegated to underlying infrastructure, such as resource limits for user notebooks.
   In other cases, such as API rate limits, they need to be dynamically calculated and imposed by Rubin code.

   This tech note discusses the requirements and possible implementation options and describes the initial quota implementation.

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
   Fair in this context primarily means starvation-avoidance.
   Overuse of a service by one user should not make that service unavailable or unusable by other users.

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

The initial implementation supports the following quota types:

- Total CPU and memory available in the Notebook Aspect, including any additional pods created via services like Dask_.
- API rate limits for all API services.
- A flag indicating whether the user is allowed to spawn new notebooks in the Notebook Aspect.

.. _Dask: https://www.dask.org/

Additional quota types we may want to add in the future:

- Separate API quotas based on the type of request.
  In some services, we will want a separate quota for expensive requests (async TAP jobs, for example) versus cheap requests (informational queries about the status of a job).
- Disk quotas: user home directories and, separately, group-owned project directories.
  We currently don't have a mechanism for handling group-owned directories or counting the usage in user home directories.
- Size of user database tables.
- TAP query quotas based on the effort required to process the query (as opposed to API limits on the TAP service itself).

This general framework is intended to support tracking other types of quotas, but only the first three types of quotas and the emergency API are currently implemented or discussed further in this tech note.

Overall design
==============

Gafaelfawr_ will be responsible for calculating quota information.
Quota details for a user will be included in the response to ``/auth/api/v1/user-info`` for a user's token.

.. _Gafaelfawr: https://gafaelfawr.lsst.io/

Quotas are enforced by authenticated user, and all requests or resources consumed by all tokens for that user, including notebook and internal tokens, will count against the same quota.
Unauthenticated accesses will not be limited by quotas (although see :ref:`dos-protection`).

Basic API rate limiting will be done by Gafaelfawr.
The ``GafaelfawrIngress`` custom resource will declare the name of the service underlying that ingress.
Multiple ingresses may map to the same service name.
Gafaelfawr will then track the number of API requests for that service within a time interval.
If a user exceeds that limit, it will start rejecting requests with a 429 status code.

This is not as good as quotas imposed directly by the service, since it means all API requests going through that ingress, whether lightweight informational requests or heavyweight data retrieval requests, count the same against the user's quota.
However, it allows easy deployment of basic quota support to every service.
This is an 80% solution; more sophisticated quotas can be added later if needed.

Services that need to apply quotas directly can request a delegated token (possibly with no scopes if they don't need a delegated token for anything else) and then use that token to retrieve quota information via ``/auth/api/v1/user-info``.
The Nublado controller will use that mechanism to get the user's Notebook Aspect quota.

Quota settings
==============

For the initial implementation, we will add quota information to the Gafaelfawr configuration via the Helm chart.

Quota configuration
-------------------

The data structure for the quota information will look something like this:

.. code-block:: yaml

   quotas:
     bypass:
       - g_admins
     default:
       api:
         datalinker: 500
         hips: 2000
         tap: 500
         vo-cutouts: 100
       notebook:
         cpu: 9
         memory: 27
     groups:
       g_developers:
         api:
           datalinker: 500
       g_restricted:
         notebook:
           cpu: 0
           memory: 0
           spawn: false

The ``bypass`` key lists groups whose members are exempt from all quotas.
The ``default`` key establishes default quotas for every user.
The ``groups`` key provides additional quotas to particular groups.
These quotas are additive, so in the above case a user who is a member of the ``g_developers`` group would have a quota of 1000 queries per 15 minutes for the ``datalinker`` service.
The exception is the ``notebook.spawn`` flag, which defaults to true but can be set to false for a particular group, which prevents that group from spawning new notebooks in the Notebook Aspect.

API quotas are in requests per fifteen minutes.
This is an awkward interval, but (as discussed in :ref:`rate-limiting`) the interval is also the length of time that the user will be blocked from accessing the service.
One minute seems too short, and one hour (used by GitHub) seems too long.
More sophisticated intervals may be added later.

The keys under ``api`` are the names of the services, as configured by the ``config.service`` key of the ``GafaelfawrIngress`` Kubernetes resource.

A given API service does not have to have a quota.
If no quota is configured, the quota for all users is unlimited and requests won't be tracked.
To disallow access, set an explicit quota of 0.

Notebook Aspect quotas are in CPU equivalents and GiB of memory.

Quota overrides
---------------

Emergency override information will be stored in the Gafaelfawr Redis under the key ``quota-override``.

The value of the key will be a JSON document whose structure matches the quota configuration.
For example:

.. code-block:: json

   {
       "bypass": [
           "g_admins"
       ],
       "default": {
           "notebook": {
               "spawn": false,
               "cpu": 4,
               "memory": 16
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
   }

Unlike the way group quotas are handled, quota overrides are not additive.
If the override generates a quota rule for a given action, that replaces the default quota.
So, for example, if the above override were in place, all users would have a quota of 10 queries per 15 minutes for the datalinker API, including members of ``g_developers`` who normally get an extra 500 queries per 15 minutes.
Similarly, all users will not be allowed to spawn new notebooks in the Notebook Aspect, and members of the ``g_users`` group would only have a quota of 10 for the vo-cutouts API.
The exception is members of the ``g_admins`` group, who will not have any quotas applied to their usage.

In the initial implementation, Gafaelfawr does not cache the quota override information and will try to retrieve it from Redis for every request potentially affected by quotas.
We will see if that creates a performance problem and add in-memory caching if it does.

Quota override API
^^^^^^^^^^^^^^^^^^

There are three Gafaelfawr APIs to get and set the quota overrides:

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

Quota override UI
^^^^^^^^^^^^^^^^^

This approach to quota overrides has a major drawback: the YAML in Phalanx is not the single source of truth for quota.
One can see a quota configuration in Phalanx and expect it to be applied, but the cluster may be applying a different quota because an override exists.

We considered instead storing the quota information only in Redis or only in the Phalanx configuration YAML (see :ref:`override-options`), but rejected those approaches for other reasons.

Instead, to raise the visibility of a quota override, we plan to use Semaphore (see :sqr:`060`) plus Squareone_ to add user notifications if a quota override exists.
This will make it obvious that special throttling rules are in place, and therefore the quotas found in Phalanx are not being applied verbatim.
(Eventually we may add a link, visible only to admins, from the banner to a UI to change or delete the quota overrides.)
This is not yet implemented.

.. _Squareone: https://github.com/lsst-sqre/squareone

Quota enforcement
=================

API
---

The ``/auth/api/v1/user-info`` route has been extended to add quota information.
The response looks like this:

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

The Nublado controller displays an error instead of the spawner form if the user is not allowed to create notebooks, either because the ``spawn`` flag is set to false or because none of the configured notebook sizes are equal to or smaller than the user's CPU and memory quotas.
This restriction is also applied on the underlying notebook creation API.

.. _rate-limiting:

Rate limiting
-------------

API quotas are enforced directly by Gafaelfawr.

``GafaelfawrIngress`` resources have a configuration key, ``config.service``, which corresponds to a ``service`` parameter to the ``/ingress/auth`` route.
Rate limiting will then be done if and only if there is an API quota for a service whose name matches the ``service`` parameter.

Because API quota usage is enforced via Gafaelfawr, ingresses that enable authentication caching via the ``config.authCacheDuration`` setting cannot correctly calculate quotas.
Only the uncached requests will count against the quota.
APIs that need to impose quotas should therefore not use ``config.authCacheDuration``, or must implement quotas directly.

Since there may be multiple Gafaelfawr pods running, and rate limits shouldn't vary based on which pod a given request is assigned to, the data for quota enforcement will be stored in Redis rather than in memory in each pod.
Gafaelfawr's current Redis is used to store tokens, which are valuable data that needs to be persisted to disk and backed up, and for which writes are relatively rare.
The quota tracking data requires huge numbers of writes but is not valuable and does not need to be persisted.
It therefore uses a second Redis instance for quota tracking that is in-memory only with no persistent storage.

The rate limiting will be done using limits_.

.. _limits: https://limits.readthedocs.io/en/stable/index.html

The rate limiting algorithm is fixed window.
This means that the user will be allowed their quota of requests within a window of time (15 minutes).
At the end of that window, their quota will reset and they'll get their full quota of requests again.
There are more complex algorithms that are better at smoothing out load (sliding window, for instance), but fixed window is easy to explain and reason about, is extremely fast and cheap to represent in Redis, and matches the way GitHub does rate limiting.

Each authorization request to Gafaelfawr increments the usage and rejects the request if an API quota applies and has been exceeded.
Rejections result in an HTTP 429 error (via a somewhat complicated process due to limitations of NGINX ``auth_subrequest`` handlers).
The 429 response will include a ``Retry-After`` header (see `Retry-After`_).

.. _Retry-After: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Retry-After

Both successful and failed responses will also include ``X-RateLimit-Limit``, ``X-RateLimit-Remaining``, ``X-RateLimit-Used``, ``X-RateLimit-Resource``, and ``X-RateLimit-Reset`` headers.
These have the same meaning as the `GitHub rate limiting headers <https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api?apiVersion=2022-11-28#checking-the-status-of-your-rate-limit>`__.

Eventually we may wish to switch to standardized rate limit headers, but that process is currently at the `Internet-Draft phase <https://datatracker.ietf.org/doc/html/draft-ietf-httpapi-ratelimit-headers>`__ and it's not clear when it will be completed.

Metrics
=======

Rate limit information is logged as part of the log message for each authentication request to Gafaelfawr.
The rate limit usage is included in the metrics event logged for the authentication request.

Rejections of API requests due to an exceeded rate limit will be logged as a metrics event.

Ideally, Gafaelfawr should also log metrics for how many users are being rate-limited and how many users have reached 50% or 75% of a rate limit.
This work is not yet done.

.. _dos-protection:

Denial of service protection
============================

This rate limiting system is intended to fairly share resources among non-malicious users issuing a normal rate of API requests.
Each request, even if rate-limited, requires processing by NGINX, an auth subrequest to Gafaelfawr, and processing by Gafaelfawr, including at least two Redis reads, one write, and often an LDAP lookup.
This means NGINX and Gafaelfawr could still be overloaded by higher quantities of traffic, such as runaway processes in tight loops or an intentional denial of service attack.

Fully defending against denial of service attacks is outside the scope of the Rubin Science Platform and not something we can reasonably expect to do.
We can, however, apply sanity limits on requests at the NGINX level to protect against being overwhelmed by accidents external to the cluster.

This can be done with `ingress-nginx annotations <https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#rate-limiting>`__, normally managed via ``GafaelfawrIngress``.
This rate limiting can only be done by IP address, not by user.
The NGINX rate limit should be higher than the quota of any given user, since it will be applied to every user and may apply to multiple users at the same time if they share an outbound IP address.

These rate limits must either be set high enough to allow for expected levels of traffic from in-cluster services that are making requests on the user's behalf, such as the Portal Aspect, or in-cluster services should be excluded from the rate limiting using ``nginx.ingress.kubernetes.io/limit-whitelist``.

Options considered
==================

.. _override-options:

Options for quota overrides
---------------------------

The original plan had been to store quota information in the database and provide an API and eventually a UI for updating it.
However, this is a bit awkward (and different than other Science Platform services) for bootstrapping.
A newly-installed cluster, or one where Gafaelfawr's storage was reset for some reason, would have no quotas until they were added through an API or eventual UI.
The quotas would also be invisible outside of the API or UI, unlike other deployment configuration, which is visible in Phalanx.

Storing the configuration in YAML makes it more visible and easier to edit in the normal case where no overrides are in place.
It does make updates more intrusive, since they require a Gafaelfawr rolling restart, but we don't believe we'll be updating the base quotas frequently.
The YAML configuration approach is also simpler and easier to implement.

Given that decision, we had to decide how to handle overrides.
The simplest approach would be to not support overrides and require updating the Phalanx configuration, but then, during an emergency, we would have to change the Gafaelfawr configuration and restart Gafaelfawr, which may be dangerous or undesirable under heavy load.
Being able to selectively override the normal configuration in Redis allows us to provide an API to change this on the fly, requiring only that Gafaelfawr be responsive.

Redis was chosen over PostgreSQL as the place to store quota overrides, since Redis is much faster to query.

This unfortunately means that in the (hopefully rare) case when special quota overrides are in place, the Phalanx configuration is deceptive and the quotas applied on the cluster don't match what's written in the configuration, even when the services show as up-to-date.
This creates the risk of leaving overrides in place longer than intended, and of confusion and frustrated debugging.

To address those concerns, we plan to tie quota overrides into a banner notification so that it will be obvious to anyone using the cluster that it is being temporarily throttled, and therefore the normal quota configuration may be overridden.
This is not yet implemented.

Options for rate limit configuration
------------------------------------

The rate-limit configuration for APIs is unsatisfying in both syntax and in semantics.
For syntax, ideally it would be specified as ``<count>/<time>`` so that both the number of requests and the time interval could be given.
But this makes the logic of adding in group quotas more complicated and confusing since they may use different time intervals.

For semantics, ideally we should only count "expensive" API calls, such as requesting a cutout or performing a TAP query, and not count "cheap" API calls, such as asking for the status of a job.
This in theory could be done via complicated rules in the ingress specifying how to match the URL and verb patterns of complex queries, but in practice that seems hard to maintain.
Alternately, we could assume that all ``GET`` requests are cheap and all requests with other verbs are expensive, but unfortunately IVOA standards require some expensive queries be accesible via ``GET``.

The current approach is the simplest and provides a general facility to impose basic rate-limits on anything, so we're going to start with it and see if it's adequate in practice.
If not, we may need to move more quota checking from Gafaelfawr to the separate services.

Options for rate limit implementation
-------------------------------------

Quota information could be included in structured form in an HTTP request header rather than requiring an API call, but we've moved away from that pattern elsewhere since the API call pattern is simpler and more straightforward.
The primary advantage of HTTP headers is to optimize away the API call to Gafaelfawr and the extra overhead involved in creating a delegated token, but we are avoiding premature optimization until we have evidence it is a problem.

There are two basic ways to respond to a user hitting a rate limit: delay the request until the rate limit allows it, or reject the request.
We've chosen to reject the request, since delaying it requires queuing it on the server, which adds load to a potentially already-overloaded server and also may create complex timeout issues that are hard to debug.

The drawback of rejecting the request is that it may produce failures in long-running processes when one of their underlying requests is rate-limited.
This will place the onus on the user to retry rate-limited requests if needed.
We may want to add support to PyVO_ for retrying rate-limited requests automatically.

.. _PyVO: https://pyvo.readthedocs.io/en/latest/

Python rate limiting packages
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

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
Since we expect many requests to come from inside the cluster via other services such as the Portal or Notebook Aspect, and even requests from multiple users outside the cluster may appear to come from a single IP address due to NAT, this cannot be used for the type of rate limiting we want to do.
We may use NGINX rate limiting for :ref:`dos-protection`.
