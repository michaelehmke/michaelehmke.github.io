---
title: "Using Browser Debugger to Reverse Engineer Angular App for Privilege Escalation"
date: 2019-09-13
tags: [bug bounty, web application]
categories: [hacking]
author_profile: false
---

This post will show how the debugger in a browser's developer tools may be used to help reverse engineer the unexecuted javascript of a modern web application built on top of a popular single-page application framework such as Angular. Since rendering decisions by SPA apps are contained within the javascript code already resident in the page, these techniques can be used to elicit the app to expose details and features of the UI that would not be rendered otherwise.

The vulnerability in this post was found during a bug bounty that has not yet been disclosed, and all code, URLs, and otherwise revealing details have been rewritten/ replaced with example data.

I started testing the web application for a company that we'll call SecureBank by creating a test account on the site securebank.com. This gave me the ability to login but provided access to little else. I was essentially welcomed with a banner that included my user name `haxxor` but besides that and the logout button, the page was empty - no links, no buttons, no fields for input. 

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/blank_welcome_page_large.png" alt="empty welcome page" class="left-align shadow" style="width:calc(1310px * .66667)">

Viewing the source gave hints that it was an Angular app, but nothing useful beyond that. Proxying the requests with Burp Suite revealed some API calls being made in the background, just from the process of logging in:

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/requests_sequence_large.png" alt="sequence of requests" class="left-align shadow" style="width:calc(1140px * .66667)">

This was plenty to get started with. I started off messing with the `/api/userAdmin/status` endpoint which returned the date of the last login of the user if within the past month. I thought that this might be vulnerable to an Insecure Direct Object Reference attack and used Burp Intruder to brute-force the `SM_USER` query parameter with a list of usernames. 10 minutes and 18,000 attempts later, I discovered 40+ valid users which had logged in recently, including a particular user we'll call `admin`, which we'll revisit later.

At this point, `/api/user/getTransactions` looked promising but it basically just returned back an indication of success or failure. Moving on to the `/api/userAdmin/permissions` endpoint, I found something interesting. When providing my username as the value for the query parameters `sso` and `SM_USER`, an empty JSON object was returned; probably indicating that I had no permissions, which wasn't surprising given my blank welcome screen. When trying the admin account through Burp Repeater however (and only necessarily for the `sso` param), I received the following JSON response:

```json
{
    "admin":[
        "USER_ADMIN",
        "EDIT_FINANCIAL",
        "WIRE_FUNDS",
        "OPEN_ACCOUNT",
        "CLOSE_ACCOUNT",
        "TRANSER",
        "RECEIVE",
        "MODIFY_PERMISSIONS",
        "OVERRIDE_CHECKS",
        "DEFAULT"
    ]
}
```

The permissions for the admin user were returned, which provides useful information, but doesn't grant me any extra abilities or anything just by receiving that response. While this is starting to confirm a weakness in the access controls placed on the app's API endpoints, this would be considered an informational finding or of low risk at best. With only two known endpoints left to test, I find that `/api/auth` was just the endpoint that was used to login with my credentials. But then I again find that `/api/user/login` can be used to pull data pertaining to another user. This time there was no parameter in the URL to modify, but I was able to specify who to pull data for by replacing `haxxor` with `admin` as the value for the HTTP header `SM_USER`. 

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/burp_api-user-login_admin_large.png" alt="request from burp with SM_SESSION modified to admin" class="left-align shadow" style="width:calc(1107px * .66667)">

The response to this request returned a JSON object of the admin user revealing data that includes the user's name, numerical userid, contacts, roles, permissions, etc. This revealed even more information than `/api/userAdmin/permissions`, but the risk still seemed relatively low so I wanted to take it further somehow. There were no additional endpoints that were captured by Burp at this point, but I knew that the application made those requests for a reason, and was processing the JSON responses to fulfill some logic. I was specifically interested in how the JSON for the permissions was being used by the app's user interface, because if I could determine what other functionality the interface provides, then I can test those parts of the app. 

Now, to jump into the Angular code in Firefox's debugger. 

<h2>Stepping Inside the Code</h2>

<h3>Manipulating /api/user/login</h3>

The endpoint `/api/user/login` appeared more interesting to me since it returned not only more data, but also included the data that was returned from `/api/userAdmin/permissions`. Searching for the string `/api/user/login` in the app's main javascript file takes us to the location where the API call is made. I set a breakpoint there to see the state of the app at that point:

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/debug_api_user_login_large.png" alt="/api/user/login breakpoint" class="left-align shadow" style="width:calc(1324px * .66667)">

After reloading the page, the breakpoint is hit. By the time execution gets to this location, note that the SSO in the local `e` variable is already set:

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/e_haxxor_large.png" alt="e variable set to haxxor" class="left-align shadow" style="width:calc(384px * .66667)">

Stepping into the `this.http.get()` function at line 7514, I set another breakpoint:

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/debug_getHeaders_large.png" alt="breakpoint set at call to getHeaders()" class="left-align shadow" style="width:calc(892px * .66667)">

Checking the value of the local var `e` here confirms the URL for the request we're currently looking at:

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/e_url_large.png" alt="e variable set to /api/user/login url" class="left-align shadow" style="width:calc(356px * .66667)">

On line 4596, it looks like it's getting and setting the HTTP headers for the call to that endpoint. Earlier, we had modified the HTTP header `SM_USER` in the request to this endpoint to elicit data for the admin user. This returned interesting data but its implications were limited. If we do this again, but  cause the app's code in the browser to make that request itself, rather than sending an out-of-band request through Burp, it may render the UI differently if there is anything that relies on that data.

To do this, we'll use the debugger to modify the value of the `SM_USER` header right after it gets set. Stepping into the getHeaders() function shows where this happens, so another breakpoint is placed here:

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/debug_set_headers_large.png" alt="breakpoint set where headers are set locally" class="left-align shadow" style="width:calc(1120px * .66667)">

The first part of the return statement sets the `SM_USER` header to the value at `this._loginSvc.authorizationResponses['SM_USER']`. Viewing the local variables shows that the value there is equal to `"haxxor"`.

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/var_authResponse_smuser_haxxor_large.png" alt="SM_USER set to haxxor in local var" class="left-align shadow" style="width:calc(936px * .66667)">

In the console, I change that value to `"admin"`:

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/set_smuser_admin_large.png" alt="set smuser to admin in console" class="left-align shadow" style="width:calc(991px * .66667)">

Now the code will use my modified value and assign it to `SM_USER` accordingly. After stepping over the assignments, we can check the local values that will be returned for the headers:

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/e_smuser_admin_large.png" alt="smuser set to admin in local var" class="left-align shadow" style="width:calc(463px * .66667)">

Step out of both of the previous functions to return to the code block where the API call was made:

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/debug_7514_large.png" alt="return two functions up" class="left-align shadow" style="width:calc(1324px * .66667)">

We can see in the console, that the call to `/api/user/login` had just been made, and with our modifed value for `SM_USER`:

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/req_api-user-login.png" alt="request showing SM_USER set to 'admin'" class="left-align shadow" style="width:calc(1063px * .66667)">

Viewing the value of the local var `e` here, we can see the JSON object that was given as the response to this request:

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/e_resp_admin_large.png" alt="value of the e variable showing data for admin" class="left-align shadow" style="width:calc(574px * .66667)">

We can see that it has been populated with data pertaining to the admin user, including the user's permissions. The client will use this data going forward, so now that we have influenced it to accept this other user's information as my own, I deactivate all breakpoints and resume execution.

I was hoping for this to effectively render the UI as the user I was attempting to impersonate but once the page fully loaded, the only change to the interface was reflected in the banner as it now welcomed `admin` instead of my username.

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/blank_admin_welcome_page_large.png" alt="blank admin welcome page" class="left-align shadow" style="width:calc(1310px * .66667)">

This single change was promising however, as it showed that the interface would be influenced by data that was returned by requests that we could manipulate. It also wasn't likely that the admin user would be presented with a blank page after legitimately logging in, given the relatively high privileges of that account that we saw earlier. I just had to find the right data that needed to be modified to reveal the full UI.

<h3>Trying again with /api/userAdmin/permissions</h3>

Looking back over the requests made during the login process:

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/requests_sequence_large.png" alt="sequence of requests" class="left-align shadow" style="width:calc(1140px * .66667)">

The call to `/api/userAdmin/permissions` stands out since it also provides data about permissions in its response. This seems redundant since the resopnse to the previous call we looked at contained this same information, but maybe the app processes it differently. When we analyzed this endpoint earlier, we only had to change the `sso` query parameter to pull data for other users:

```
/api/userAdmin/permissions?sso=haxxor&SM_USER=haxxor
```

Our goal now is similar to that of earlier; we just need to load the page in a debugging session and manipulate the `sso` query parameter to equal `admin` before that request is made.

We start by searching for the string `/userAdmin/permissions` in the app's main javascript file and setting a breakpoint where the `HttpParams` are getting set:

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/debug_userAdmin_permissions_large.png" alt="breakpoint in function where call to /userAdmin/permissions is requested" class="left-align shadow" style="width:calc(1258px * .66667)">

We'll go ahead and reload the page so that execution of the app restarts and pauses at our breakpoint. Viewing the local var `e`, which the `ssoId` parameter later gets set to, shows the current value:

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/e_haxxor_for_ssoid_large.png" alt="e is equal to 'haxxor'" class="left-align shadow" style="width:calc(384px * .66667)">

Using the console, we'll modify the value of `e` to be `"admin"`:

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/console_set_e_admin.png" alt="setting e to admin in console" class="left-align shadow" style="width:calc(231px * .66667)">

Stepping over in the code shows the new value of `e`:

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/e_set_to_admin_large.png" alt="value of e is admin" class="left-align shadow" style="width:calc(370px * .66667)">

The code was stepped through until the call to the endpoint had been made. The request sent can be seen in the console:

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/requests_permissions_large.png" alt="ssoid param set to admin in request shown" class="left-align shadow" style="width:calc(1187px * .66667)">

We can see that the `ssoid` parameter was successfully set to `admin` for the request. Viewing the local var `e` shows the JSON response of the admin's permissions being stored, as expected:

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/admin_perms_received_large.png" alt="admin permissions shown in local var e" class="left-align shadow" style="width:calc(482px * .66667)">

Now that the client has accepted this data, we resume execution until our old breakpoints are hit. When execution pauses at the call to `/api/user/login`, we perform the same data manipulation as earlier to make this request under the guise of the admin user.

After completing the full execution, I view the loaded web page hoping for new UI elements but still nothing. This time there are no new changes, only the same banner welcoming the admin user from the intitial attempt that we just repeated.

<h3>Bypassing client-side validations</h3>

After some analysis of the code, I learn that there is some client-side validation done on the ssoId retured in the response to `/api/userAdmin/permissions`. The validation ensures that it matches the actual Id of the user, which was set at some point prior. In the JSON response below, the returned ssoId `admin` was expected to match my actual ssoId of `haxxor`:

```json
{
    "admin":[
        "USER_ADMIN",
        "EDIT_FINANCIAL",
        "WIRE_FUNDS",
        "OPEN_ACCOUNT",
        "CLOSE_ACCOUNT",
        "TRANSER",
        "RECEIVE",
        "MODIFY_PERMISSIONS",
        "OVERRIDE_CHECKS",
        "DEFAULT"
    ]
}
```

Since there is a mismatch, the branch of code that acts on the returned permission set does not execute. To get around this, we reload the page and repeat the steps at the breakpoints for the `/api/userAdmin/permissions` endpoint up to the point where we receive the JSON response containing the permissions above. This will again be stored in the local var `e`:

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/admin_perms_received_large.png" alt="admin permissions shown in local var e" class="left-align shadow" style="width:calc(482px * .66667)">

I chose to just change the ssoID in the JSON object to match my ssoId `haxxor` rather than try to find all the places where validations might occur on this value throughout the codebase and changing them all to expect `admin`. 

To do this, we stringify the JSON object in var `e` from the console:

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/json_stringify_large.png" alt="stringified json object of admin permissions" class="left-align shadow" style="width:calc(874px * .66667)">

Then modify the ssoId to be `haxxor`:

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/json_modified_large.png" alt="json string modified from admin to haxxor" class="left-align shadow" style="width:calc(1035px * .66667)">

Turn the string back into a JSON object:

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/json_string_to_object_large.png" alt="transforming json string to json object in console" class="left-align shadow" style="width:calc(510px * .66667)">

Finalize the modification by setting `e` to our new JSON object:

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/json_assign_to_e_large.png" alt="assign json obj back to e" class="left-align shadow" style="width:calc(463px * .66667)">

Step over to confirm the value of `e`:

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/e_confirm_haxxor_large.png" alt="confirmed ssoid in json has changed from admin to haxxor" class="left-align shadow" style="width:calc(488px * .66667)">

The ssoId has successfully been changed to `haxxor` while retaining the same set of permissions. Going forward, the client should use these values.

At this point we've pulled permissions for the admin user from the `/api/userAdmin/permissions` endpoint and have modified the stored JSON to reflect my own ssoId in order to pass any validation checks. Now, we resume execution to hit our old breakpoints to manipulate the request to `/api/user/login` as we had done originally. After this has been done once again, we continue execution to completion and view the page.

Finally, the full interface for the admin user has been revealed. 

<img src="{{ site.url }}{{ site.baseurl }}/images/js_dbg_privesc/large/admin_welcome_page_large.png" alt="revealed admin welcome page" class="left-align shadow" style="width:calc(1310px * .66667)">

This effectively allowed for the impersonation of any user and provided the opportunity to fully explore the UI and discover new functionalities and endpoints of the app. Also, as hinted by the lack of access controls on the endpoints tested so far, authorization for any endpoint was missing as long as the user was authenticated, and allowed my user without any privileges to view all the data loaded into the admin's UI and trivially perform any action from the UI that the impersonated user could perform.

<h2>Takeaways</h2>

The main takeaway here for developers is to perform authorization checks on every API endpoint on the **server-side** against a unique value that identifies a user that cannot be guessed, such as a session cookie, and not a userID that may be easily enumerated. I understand that sometimes developers will intentionally neglect performing access control checks on every API call due to the potential performance overhead of doing so, but that is why it is better for security to be built-in from the start as a criteria for the application's design, rather than tacked on at the end. 

For penetration testers working on Angular applications or applications built on SPA frameworks which do much of the rendering decisions client-side; hopefully this served as a good example for how the browser's debugger can prove to be a useful tool for getting more out of the unexecuted javascript that is sitting in the page, waiting on some condition to be fulfilled to render other components.

Of course, revealing the front-end interface isn't a vulnerability in and of itself, since we have access to all of that code anyways. Causing the code to be rendered in the browser however, may be a much more effective use of time over statically reverse engineering the appropriate sets of API calls and the ordering of those calls that may be required to carry out any funcationality, especially if the code is minified.

In the case of this application, the real vulnerability lied in the lack of server-side authorization on its API endpoints; though at the start I was only aware of a few endpoints, and the implications at that point were fairly low-risk. However after discovering more endpoints after impersonating users client-side, the vulnerability became much high-risk as I was also able to show how users could be impersonated fully through actions taken against the server's endpoints.

