Django integration for HTTPolice
================================

.. highlight:: console

Django-HTTPolice is a package that integrates `HTTPolice`__ into Django 1.8+.

__ http://httpolice.readthedocs.io/en/stable/


Installation
------------

::

  $ pip install Django-HTTPolice

.. highlight:: py

This package provides :class:`django_httpolice.HTTPoliceMiddleware`.
Add it to your `MIDDLEWARE_CLASSES`, as close to the top as possible::

  MIDDLEWARE_CLASSES = [
      'django_httpolice.HTTPoliceMiddleware',
      'django.middleware.common.CommonMiddleware',
      # ...
  ]

This middleware does **nothing** until you
also set the `HTTPOLICE_ENABLE` setting to `True`.

When enabled,
the middleware checks all :ref:`exchanges <exchanges>` passing through it.
Then, there are two different ways to see the results of these checks.


Viewing the backlog
-------------------
All exchanges checked by the middleware are stored
in a global variable called the *backlog*.
By default, it holds up to 20 latest exchanges,
but you can override by setting `HTTPOLICE_BACKLOG` to a different number.

The package also provides the :func:`django_httpolice.report_view` function.
Add it to your URLconf like this::

  import django_httpolice
  
  urlpatterns = [
      # ...
      url(r'^httpolice/$', django_httpolice.report_view),
      # ...
  ]

When you start the server and open ``/httpolice/`` (or whatever URL you chose),
you will see an HTML report on all the exchanges currently in the backlog.
The **latest** exchanges are shown at the **top** of the report.

If `HTTPOLICE_ENABLE` is not `True`, the view responds with 404 (Not Found).

You can also access the backlog from your own code:
it’s in the :data:`django_httpolice.backlog` variable,
as a sequence of :class:`httpolice.Exchange` objects.


Raising on errors
-----------------
If you set the `HTTPOLICE_RAISE` setting to `True`,
then the middleware will raise a :exc:`django_httpolice.ProtocolError`
whenever a **response** is found to have any errors
(that are not :ref:`silenced <django-silence>`).

.. highlight:: console

This can be used to fail tests on errors::

  $ python manage.py test
  .E.
  ======================================================================
  ERROR: test_get_plain (example_app.test.ExampleTestCase)
  ----------------------------------------------------------------------
  Traceback (most recent call last):
    [...]
    File "[...]/django_httpolice/middleware.py", line 81, in process_response
      raise ProtocolError(exchange)
  django_httpolice.common.ProtocolError: HTTPolice found errors in this response:
  ------------ request 1 : GET /api/v1/?name=Martha&format=plain
  C 1070 No User-Agent header
  ------------ response 1 : 200 OK
  E 1038 Bad JSON body
  
  
  ----------------------------------------------------------------------
  Ran 3 tests in 0.351s
  
  FAILED (errors=1)

.. highlight:: py

The exchange is still added to the backlog.


.. _django-silence:

Silencing unwanted notices
--------------------------
To :ref:`silence <silence>` notices you don't care about,
you can use the `HTTPOLICE_SILENCE` setting::

  HTTPOLICE_SILENCE = [1070, 1110, 1194]

They will disappear from reports and will not cause `ProtocolError`.

By default, `HTTPOLICE_SILENCE` includes some notices
that are irrelevant because of Django specifics, such as `1110`__.

__ http://pythonhosted.org/HTTPolice/notices.html#1110

Of course, the ``HTTPolice-Silence`` header works, too::

  def test_unauthorized(self):
      response = self.client.get('/api/v1/products/',
                                 HTTP_HTTPOLICE_SILENCE='1194 resp')
      self.assertEqual(response.status_code, 401)
