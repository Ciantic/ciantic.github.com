=============================
Multisite approach for django
=============================

In order to implement Django instance that could serve multiple sites per instance with different users and permissions per site. I propose either of the two approaches to fix Django.

Additionally different sites should have possibility of having different media root and url, since these represent the site media. Here we assume that installed apps are same for each site, so the static url and root can be left as global.

1. Using methods ``get_request_*(request)``
===========================================


It would be **required** to implement following::

    get_request_site_id(request) 
    get_request_media_root(request) 
    get_request_media_url(request)
    
Authentication and logins per site
----------------------------------

In order to implement authentication per site there needs to be at least a another auth backend method (if not altered completly see below)::

    get_request_user(user_id, request) 

And replace the auth middleware's (``django.contrib.auth.get_user()``) auth backend ``backend.get_user(user_id)`` call with ``backend.get_request_user(user_id, request)`` if backend implements it (the backwards compatible way).

Also the ``AuthenticationForm`` and login view provided in Django are not reusable in per site authentication. It is not possible to simply subclass the ``AuthenticationForm`` since it is required to pass ``request`` object *or* site id to own auth backend in order to determine the site of request. So the calls to authentication backend should be in form of more general: ``backend.authenticate(request, username, password)`` or more restrictive ``backend.authenticate(site_id, username, password)`` to solve this problem. 

Currently there is *no way* to access request object in ``django.contrib.auth.forms.AuthenticationForm.clean()`` even though there is attribute ``request``, it is ``None`` when Django calls the ``clean()``.

2. Using middleware
===================

It is tedious to type ``get_request_*(request)`` calls since one needs them almost *always* in *all* views if apps were to use sites.
    
So simply add the ``site_id``, ``media_root``, ``media_url`` to the request object in middleware (just like the ``request.urlconf`` is currently added). This is handy since it only needs one cache lookup, whereas methods require fetch from cache most of the time. Behind this middleware approach there has to be methods just like in first proposal, but user would not need to type them all the time.

(The auth middleware still needs to be fixed as in first proposal.)

In either of the proposals the per site ``media_url`` and ``media_root`` could be accessed in templates just like before since template context processor would add them by request. Although these will considerably complicate things with model fields like ``FileField`` which now would require to take in the request object always.


Altering the auth backend role. (optional)
==========================================
I think this module level ``django.contrib.auth.get_user()``, ``login()`` is currently wrong way to do things in the first place (and they are confusingly named for their behavior, see below), it cuts out the authentication backend all together.

It would be simpler to imagine this login / get_user as following: 

- Saving the user to ``request.session``

  (currently handled by the module level ``login()``) 
  
- Loading user from ``request.session`` (all requests after login) 

  (currently handled by the module level ``get_user()``, and ``backend.get_user()``)

There is no simple way one could define own behavior for this saving and loading the user to ``request.session`` in auth backends. It seems like these two should be overridable in auth backend. 

If there were a way to define saving and loading user one could do cool stuff like: 
    
1. per site login system (saving site_id of request to session during saving and fetching it from session during loading) 
2. during login caching of user permissions to session (no need to hit database after login for permissions!) 
    
One could define the saving and loading of user to session in authentication backend with simple methods like:

- ``backend.save_user(request, user)`` -> bool 
- ``backend.load_user(request)`` -> user object 

(The loading could be named ``get_user`` since it is like loading, but it clashes with old behavior that just gets the user from ``user_id``. Keeping the backwards compatibility is a good thing I suppose better name would be ``backend.get_request_user(request)``. Above I used word *load* for the sake of the consistency with my example.)

Then for backwards compatibility we could do: 

- ``django.contrib.auth.login`` would become the caller for ``backend.save_user(request, user)`` of course the ``login()`` could still use the backend_session_key and user_id there
- ``django.contrib.auth.get_user`` would become the caller for ``backend.load_user(request)``

Later on these module level functions could be renamed by their *actual role*, like to ``save_to_request(request, user)`` and ``get_request_user(request)`` or something like that. (*Save to request* is a bit misleading and *save to session* is too restraining for the auth backend (auth backend needs the request object))