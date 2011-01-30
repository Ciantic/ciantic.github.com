=============================
Multisite approach for django
=============================

In order to implement Django instance that could serve multiple sites per instance with different users and permissions per site. I propose either of the two approaches to fix Django.

1. Using methods ``get_request_*(request)``
===========================================


It would be **required** to implement following::

    get_request_site_id(request) 
    get_request_media_root(request) 
    get_request_media_url(request)

And at least a another backend method (if not altered completly see below)::

    get_user_request(user_id, request=None) 

And replace the auth middleware's (``django.contrib.auth.get_user()``) auth backend ```get_user()`` call with ``get_user_request()`` if backend implements it (the backwards compatible way).

2. Using middleware
===================

It is tedious to type ``get_request_*(request)`` calls after all one needs them almost *always in all* views if apps were to use sites.
    
So simply add the ``site_id``, ``media_root``, ``media_url`` to the request object in middleware (just like the ``request.urlconf`` is currently added). This is handy since it only needs one cache lookup, whereas methods require fetch from cache most of the time. Behind this middleware there had to be method just like in first proposal, but user would not need to type them all the time.

The auth middleware still needs to be fixed as in 1. proposal.


Altering the auth backend role. (optional)
==========================================
I think this module level ``django.contrib.auth.get_user()``, ``login()`` is currently wrong way to do things in the first place (and they are confusingly named for their behavior, see below), it cuts out the authentication backend all together.

It would be simpler to imagine this login / get_user as following: 

- Saving the request's user to request.session 

  (currently handled by the module level ``login()``) 
  
- Loading user from request's session (all requests after login) 

  (currently handled by the module level ``get_user()``, and ``backend.get_user()``)

There is no simple way one could define own behavior for this saving and loading the user to request.session in auth backends. It seems like the thing would be handy to be easily replacaple in auth backend. 

If there were a way to define saving and loading user one could do cool stuff like: 
    
1. per site login system (saving site_id to session during saving and fetching it from session during loading) 
2. during login caching of user permissions to session (no need to hit database after login for permissions!) 
    
One could define the saving and loading of user to session in authentication backend with simple methods like::

    authbackend.save(request, user) -> bool 
    authbackend.load(request) -> user object 

Then for backwards compatibility we could do (since removing ``login`` is out of question or renaming to save and get/load): 

- ``django.contrib.auth.login`` would become the caller for backend.save(request, user) of course the login() could still use the backend_session_key and user_id there
- ``django.contrib.auth.get_user`` would become the caller for backend.load(request) 