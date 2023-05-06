# CSRF Protection
- [Introduction](#csrf-introduction)
- [Preventing CSRF Requests](#preventing-csrf-requests)
    - [Excluding URIs](#csrf-excluding-uris)


- [X-CSRF-Token](#csrf-x-csrf-token)
- [X-XSRF-Token](#csrf-x-xsrf-token)

## [Introduction](#csrf-introduction)
Cross-site request forgeries are a type of malicious exploit whereby unauthorized commands are performed on behalf of an authenticated user. Thankfully, Laravel makes it easy to protect your application from [cross-site request forgery](https://en.wikipedia.org/wiki/Cross-site_request_forgery) (CSRF) attacks.

#### [An Explanation Of The Vulnerability](#csrf-explanation)
In case you're not familiar with cross-site request forgeries, let's discuss an example of how this vulnerability can be exploited. Imagine your application has a `/user/email` route that accepts a `POST` request to change the authenticated user's email address. Most likely, this route expects an `email` input field to contain the email address the user would like to begin using.

Without CSRF protection, a malicious website could create an HTML form that points to your application's `/user/email` route and submits the malicious user's own email address:

```blade	
<formaction="https://your-application.com/user/email"method="POST">
<inputtype="email"value="[[email protected]](/cdn-cgi/l/email-protection)">
</form>
 
<script>
 document.forms[0].submit();
</script>
```
If the malicious website automatically submits the form when the page is loaded, the malicious user only needs to lure an unsuspecting user of your application to visit their website and their email address will be changed in your application.

To prevent this vulnerability, we need to inspect every incoming `POST`, `PUT`, `PATCH`, or `DELETE` request for a secret session value that the malicious application is unable to access.

## [Preventing CSRF Requests](#preventing-csrf-requests)
Laravel automatically generates a CSRF "token" for each active [user session](/docs/master/session) managed by the application. This token is used to verify that the authenticated user is the person actually making the requests to the application. Since this token is stored in the user's session and changes each time the session is regenerated, a malicious application is unable to access it.

The current session's CSRF token can be accessed via the request's session or via the `csrf_token` helper function:

```php	
use Illuminate\Http\Request;
 
Route::get('/token', function(Request$request) {
$token=$request->session()->token();
 
$token=csrf_token();
 
// ...
});
```
Anytime you define a "POST", "PUT", "PATCH", or "DELETE" HTML form in your application, you should include a hidden CSRF `_token` field in the form so that the CSRF protection middleware can validate the request. For convenience, you may use the `@csrf` Blade directive to generate the hidden token input field:

```blade	
<formmethod="POST"action="/profile">
@csrf
 
<!-- Equivalent to... -->
<inputtype="hidden"name="_token"value="{{csrf_token()}}" />
</form>
```
The `App\Http\Middleware\VerifyCsrfToken`[middleware](/docs/master/middleware), which is included in the `web` middleware group by default, will automatically verify that the token in the request input matches the token stored in the session. When these two tokens match, we know that the authenticated user is the one initiating the request.

### [CSRF Tokens & SPAs](#csrf-tokens-and-spas)
If you are building a SPA that is utilizing Laravel as an API backend, you should consult the [Laravel Sanctum documentation](/docs/master/sanctum) for information on authenticating with your API and protecting against CSRF vulnerabilities.

### [Excluding URIs From CSRF Protection](#csrf-excluding-uris)
Sometimes you may wish to exclude a set of URIs from CSRF protection. For example, if you are using [Stripe](https://stripe.com) to process payments and are utilizing their webhook system, you will need to exclude your Stripe webhook handler route from CSRF protection since Stripe will not know what CSRF token to send to your routes.

Typically, you should place these kinds of routes outside of the `web` middleware group that the `App\Providers\RouteServiceProvider` applies to all routes in the `routes/web.php` file. However, you may also exclude the routes by adding their URIs to the `$except` property of the `VerifyCsrfToken` middleware:

```php	
<?php
 
namespace App\Http\Middleware;
 
use Illuminate\Foundation\Http\Middleware\VerifyCsrfTokenas Middleware;
 
classVerifyCsrfTokenextendsMiddleware
{
/**
 * The URIs that should be excluded from CSRF verification.
 *
 * @vararray
*/
protected$except= [
'stripe/*',
'http://example.com/foo/bar',
'http://example.com/foo/*',
 ];
}
```
> **Note**  
>  For convenience, the CSRF middleware is automatically disabled for all routes when [running tests](/docs/master/testing).

## [X-CSRF-TOKEN](#csrf-x-csrf-token)
In addition to checking for the CSRF token as a POST parameter, the `App\Http\Middleware\VerifyCsrfToken` middleware will also check for the `X-CSRF-TOKEN` request header. You could, for example, store the token in an HTML `meta` tag:

```blade	
<metaname="csrf-token"content="{{csrf_token()}}">
```
Then, you can instruct a library like jQuery to automatically add the token to all request headers. This provides simple, convenient CSRF protection for your AJAX based applications using legacy JavaScript technology:

```js	
$.ajaxSetup({
 headers: {
'X-CSRF-TOKEN': $('meta[name="csrf-token"]').attr('content')
 }
});
```
## [X-XSRF-TOKEN](#csrf-x-xsrf-token)
Laravel stores the current CSRF token in an encrypted `XSRF-TOKEN` cookie that is included with each response generated by the framework. You can use the cookie value to set the `X-XSRF-TOKEN` request header.

This cookie is primarily sent as a developer convenience since some JavaScript frameworks and libraries, like Angular and Axios, automatically place its value in the `X-XSRF-TOKEN` header on same-origin requests.

> **Note**  
>  By default, the `resources/js/bootstrap.js` file includes the Axios HTTP library which will automatically send the `X-XSRF-TOKEN` header for you.
