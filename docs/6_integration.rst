.. _integration:

Integration
===========

Axes is intended to be pluggable and usable with 3rd party authentication solutions.

This document describes the integration with some commonly used 3rd party packages
such as Django Allauth and Django REST Framework.


Integrating with Django Allauth
-------------------------------

Axes relies on having login information stored under ``AXES_USERNAME_FORM_FIELD`` key
both in ``request.POST`` and in ``credentials`` dict passed to
``user_login_failed`` signal.

This is not the case with Allauth. Allauth always uses the ``login`` key in post POST data
but it becomes ``username`` key in ``credentials`` dict in signal handler.

To overcome this you need to use custom login form that duplicates the value
of ``username`` key under a ``login`` key in that dict and set ``AXES_USERNAME_FORM_FIELD = 'login'``.

You also need to decorate ``dispatch()`` and ``form_invalid()`` methods of the Allauth login view.

``settings.py``::

    AXES_USERNAME_FORM_FIELD = 'login'

``example/forms.py``::

    from allauth.account.forms import LoginForm

    class AxesLoginForm(LoginForm):
        """
        Extended login form class that supplied the
        user credentials for Axes compatibility.
        """

        def user_credentials(self):
            credentials = super(AllauthCompatLoginForm, self).user_credentials()
            credentials['login'] = credentials.get('email') or credentials.get('username')
            return credentials

``example/urls.py``::

    from django.utils.decorators import method_decorator

    from allauth.account.views import LoginView

    from axes.decorators import axes_dispatch
    from axes.decorators import axes_form_invalid

    from example.forms import AxesLoginForm

    LoginView.dispatch = method_decorator(axes_dispatch)(LoginView.dispatch)
    LoginView.form_invalid = method_decorator(axes_form_invalid)(LoginView.form_invalid)

    urlpatterns = [
        # Override allauth default login view with a patched view
        url(r'^accounts/login/$', LoginView.as_view(form_class=AllauthCompatLoginForm), name='account_login'),
        url(r'^accounts/', include('allauth.urls')),
    ]


Integrating with Django REST Framework
--------------------------------------

Modern versions of Django REST Framework after 3.7.0 work normally with Axes.

Django REST Framework versions prior to
[3.7.0](https://github.com/encode/django-rest-framework/commit/161dc2df2ccecc5307cdbc05ef6159afd614189e)
require the request object to be passed for authentication.

``example/authentication.py``::

    from rest_framework.authentication import BasicAuthentication

    class AxesBasicAuthentication(BasicAuthentication):
        """
        Extended basic authentication backend class that supplies the
        request object into the authentication call for Axes compatibility.

        NOTE: This patch is only needed for DRF versions < 3.7.0.
        """

        def authenticate(self, request):
            # NOTE: Request is added as an instance attribute in here
            self._current_request = request
            return super(AxesBasicAuthentication, self).authenticate(request)

        def authenticate_credentials(self, userid, password, request=None):
            credentials = {
                get_user_model().USERNAME_FIELD: userid,
                'password': password
            }

            # NOTE: Request is added as an argument to the authenticate call here
            user = authenticate(request=request or self._current_request, **credentials)

            if user is None:
                raise exceptions.AuthenticationFailed(_('Invalid username/password.'))

            if not user.is_active:
                raise exceptions.AuthenticationFailed(_('User inactive or deleted.'))

            return (user, None)


Integrating with Django Simple Captcha
--------------------------------------

Axes supports Captcha with the Django Simple Captcha package in the following manner.

``settings.py``::

    AXES_LOCKOUT_URL = '/locked'

``example/urls.py``::

    url(r'^locked/$', locked_out, name='locked_out'),

``example/forms.py``::

    class AxesCaptchaForm(forms.Form):
        captcha = CaptchaField()

``example/views.py``::

    from example.forms import AxesCaptchaForm

    def locked_out(request):
        if request.POST:
            form = AxesCaptchaForm(request.POST)
            if form.is_valid():
                ip = get_ip_address_from_request(request)
                reset(ip=ip)
                return HttpResponseRedirect(reverse_lazy('signin'))
        else:
            form = AxesCaptchaForm()

        return render_to_response('captcha.html', dict(form=form), context_instance=RequestContext(request))

``example/templates/example/captcha.html``::

    <form action="" method="post">
        {% csrf_token %}

        {{ form.captcha.errors }}
        {{ form.captcha }}

        <div class="form-actions">
            <input type="submit" value="Submit" />
        </div>
    </form>
