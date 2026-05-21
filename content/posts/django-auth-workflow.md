---
title: "Django Authentication Workflow"
date: 2014-07-13
categories: ["Django"]
tags: ["django", "auth", "python"]
slug: "django-auth-workflow"
---

## Django Authentication

The existing Django Documentation explains everything about the Django authentication process in a pretty good way. Directions include adding your back end, replacing the built-in User model and many more. However, despite this good coverage, I had certain problems with seeing the big picture of the overall process. At times when I needed to create a new authentication backend, I always had to glance through the Django source again.

In this blog entry I'll focus on the Django source code and documentation -- to be more exact, I will share my findings during reading both the source code and the documentation.

## Workflow as Pseudocode

If I play around with a typical Django app using all the defaults, the authentication process will look like this:

- The user comes to django.contrib.auth.authenticate
- In case this step is successful, the next stop is reaching django.contrib.auth.login
- The final step is the user getting redirected to the successful login page.

But if I dive deeper and present the process in a detailed way, things will be structured like this:

`django.contrib.auth.authenticate` : Traverses all of the registered backends. The backend that is responsible for the user authentication gets assigned to the current user object. Then, the user object is returned back.

The code is something like this (stripped):

```python
def authenticate(**credentials):
    """
    If the given credentials are valid, return a User object.
    """
    for backend in get_backends():
        try:
            user = backend.authenticate(**credentials)
        except TypeError:
            continue
        except PermissionDenied:
            return None
        if user is None:
            continue
        # Annotate the user object with the path of the backend.
        user.backend = "%s.%s" % (backend.__module__, backend.__class__.__name__)
        return user
```

The next step is to save the user and his corresponding backend into the session object. This can be done via the login function, present in the `django.contrib.auth`.

`django.contrib.auth.login` :

```python
def login(request, user):
    request.session[SESSION_KEY] = user.pk
    request.session[BACKEND_SESSION_KEY] = user.backend
```

Most of the code you see here is stripped in order to simplify the concept and have you understand it easier. After the user sends back a response to the whole auth/login process, the `SessionMiddleware.process_response` saves the changed session object into its database. This effectively means that next time the user accesses some of the views, the object will be loaded from this saved point.

## Checking User in Session

But what happens if the user gains access to the predefined `login_required` views after the login process? In this case, we're looking at a workflow structured like this:

- `SessionMiddleware` loads the data of the current user in the request object from the session storage
- `AuthenticationMiddleware` then loads the user from the session database and sets it as an attribute in the request object
- And finally, the user has passed the auth check successfully and he can access the requested page

Again, if I break the process down and present it in a more in-depth manner, things will be structured like this:

`django.contrib.sessions.middleware.SessionMiddleware` : Loads the session info that is related to the current user:

```python
class SessionMiddleware(object):
    def process_request(self, request):
        engine = import_module(settings.SESSION_ENGINE)
        session_key = request.COOKIES.get(settings.SESSION_COOKIE_NAME, None)
        request.session = engine.SessionStore(session_key)
```

The next step to be taken is:

`django.contrib.auth.middleware.AuthenticationMiddleware` : This part is responsible for loading the backend which was saved above (login function). It also prompts the backend `get_user` method to load the user object. It is something like:

```python
def get_user(request):
    from .models import AnonymousUser
    try:
        user_id = request.session[SESSION_KEY]
        backend_path = request.session[BACKEND_SESSION_KEY]
        assert backend_path in settings.AUTHENTICATION_BACKENDS
        backend = load_backend(backend_path)
        user = backend.get_user(user_id) or AnonymousUser()
    except (KeyError, AssertionError):
        user = AnonymousUser()
    return user

request.user = get_user(request)
```

I should note that if the user is not found either in the session store or the authentication backend, the `AnonymousUser` will be returned back. After undertaking such a step, the request object will now have a `user` attribute assigned to it. In such a case you can easily check if a certain user is authenticated by typing:

```python
if request.user.is_authenticated():
    #do some auth stuff
```

Actually, that is how `login_required` decorator is implemented. It checks for existing `user.is_authenticated()` condition on current user.

## Summary

To summarize all of this, Django caches the user's authentication backend when the login process is successful. After consecutive requests, this info gets extracted from the session store and is then used to load the user object into the current request.

Finally, the views that need to check if a user is authenticated in a valid way can do so by invoking the `is_authenticated` method of the aforementioned user.
