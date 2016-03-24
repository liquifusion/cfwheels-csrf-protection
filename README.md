# CFWheels CSRF Protection Plugin

Out of the box, CFWheels applications have a Cross-Site Request Forgery (CSRF) security vulnerability. OWASP
has an excellent [overview of CSRF][1] if you're unfamiliar with this vulnerability (or need a refresher).

This plugin helps protect against CSRF attacks by authorizing all `POST` requests against the user's session.
All `POST` requests must contain an authenticity token either via a field named `authenticityToken` (usually
provided as a hidden form field) or a request header named `X-CSRF-Token` (usually for AJAX `POST`s). `POST`ed
requests not containing this authenticity token will throw an exception named
`Wheels.InvalidAuthenticityToken`.

## Setup

Install this plugin by grabbing the zip file from the latest release in the [Releases tab][2] on GitHub,
placing it in the `plugins` folder of your CFWheels application, and reloading your app.

Next, add a call to the `protectFromForgery` controller intializer in your base
`controllers/Controller.cfc`:

```coldfusion
<cffunction name="init">
	<cfset protectFromForgery()>
</cffunction>
```

And then add a call to the `csrfMetaTags` view helper to your layout's `<head>` section:

```
<cfoutput>

<head>
	#csrfMetaTags()#
</head>

</cfoutput>
```

Your application is now CSRF-protected, given that you're good with the _Prerequisites_ listed below.

If you have an API, you'll probably want to read the _Skipping CSRF protection for APIs_ section later in
this README.

## Prerequisites

This plugin assumes that your CFWheels application only changes data within your application via HTTP `POST`,
`PUT`, `PATCH`, and `DELETE` requests. The plugin protects all of these types of requests (excluding `GET` and
`HEAD` requests).

At the time of this writing, web browser clients will only reliably perform `POST` requests, so if your
CFWheels forms are mutating data only via `POST` requests, you're good to go. If not, then you'll need to
double-check this before patching up your application with this plugin.

An example of good practice is using `startFormTag`'s default value for `method` whenever you're changing data
through a form:

```coldfusion
<cfoutput>
<!-- The default is method="post" -->
#startFormTag(route="users")#
	<!--- Etc. --->
#endFormTag()#

<!-- Or if you're outside of CFWheels helpers -->
<form action="#urlFor(route='users')#" method="post">
	<!-- Etc. -->
</form>
</cfoutput>
```

Appropriately, this plugin adds a hidden field within all of your `startFormTag`-based forms with an
`authenticityToken` that proves that a real human being is posting the form as they should be:

```html
<input type="hidden" name="authenticityToken" value="">
```

## Skipping CSRF protection for APIs

You'll likely not want CSRF protection enabled for API endpoints (which should be authenticated in some other
way like OAuth and/or via a token system anyway). You can avoid the CSRF protection by either not including it
in the API's inheritance chain or by using `protectFromForgery`'s `except` argument.

Here is an example of how to avoid `protectFromForgery` if you have your API mixed in with a traditional HTML
app:

```coldfusion
<!-- controllers/Controller.cfc -->
<cfcomponent extends="Wheels">
	<cffunction name="init">
		<cfargument name="includeForgeryProtection" type="boolean" required="false" default="true">

		<cfif arguments.includeForgeryProtection>
			<cfset protectFromForgery()>
		</cfif>
	</cffunction>
</cfcomponent>

<!-- controllers/Api.cfc -->
<cfcomponent extends="Controller">
	<cffunction name="init">
		<!-- Turn off forgery protection for all API endpoints that enxtend this controller -->
		<cfset super.init(includeForgeryProtection=false)>

		<cfset provides("json")>
	</cffunction>
</cfcomponent>

<!-- controllers/ApiUsers.cfc -->
<cfcomponent extends="Api">
	<cffunction name="init">
		<cfset super.init()>
	</cffunction>

	<cffunction name="create">
	</cffunction>
</cfcomponent>
```

## Open Issue: AJAX Calls

JavaScript calls that use AJAX to `POST` to your CFWheels app may get tripped up by CSRF Protection.

jQuery or whatever you're using for AJAX should be configured to read the `meta` tags generated by the
`csrfMetaTags` helper, and then post its value via an HTTP header named `X-CSRF-Token`.

Here is an example of configuring jQuery to do this for all AJAX calls:

```javascript
define(['jquery'], function ($) {
  var token = $('meta[name="csrf-token"]').attr('content');

  $.ajaxSetup({
    beforeSend: function (xhr) {
      xhr.setRequestHeader('X-CSRF-Token', token);
    }
  });

  return token; 
});
```

## License

The MIT License (MIT)

Copyright (c) 2016 Liquifusion Studios


[1]: https://www.owasp.org/index.php/Cross-Site_Request_Forgery_%28CSRF%29
[2]: https://github.com/liquifusion/cfwheels-csrf-protection/releases
