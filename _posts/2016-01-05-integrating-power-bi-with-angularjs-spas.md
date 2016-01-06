---
layout: post
title: Integrating Power BI with AngularJS SPAs
comments: true
---

[Power BI](https://powerbi.com) reports and dashboards can be integrated into other Web Apps using iframe embedding and the Power BI 
[REST APIs](https://powerbi.microsoft.com/en-us/documentation/powerbi-developer-overview-of-power-bi-rest-api/). [Code samples](https://github.com/Microsoft/PowerBI-CSharp/tree/master/samples/webforms)
on GitHub show how to set it up using an ASP back-end. It is also possible to skip the back-end and code everything using JavaScript on the client side,
a great match for [AngularJS](https://angularjs.org/) SPAs.

![Power BI integration]({{ site.url }}/images/pbi.jpg)

# Web App registration

First the app needs to be registered with [Azure Active Directory](https://azure.microsoft.com/en-us/services/active-directory/) (AAD): 
to do so, go to the Active Directory tab in the [Azure Portal](https://manage.windowsazure.com/), click on the Applications tab, 
and then the Add button. After filling-in the quick wizard, go to the app Configure tab and add 3 Delegated Permissions to 'Power BI Service': View users Groups, View all Reports, and View all Dashboards. Also
take a quick look at the page URL: the first GUID is your Tenant ID and the second is your Client ID. Write them down, they will be needed later.

![AAD IDs]({{ site.url }}/images/aad_ids.png)

JavaScript code running in the browser cannot be trusted, so authentication cannot rely on API keys and the OAuth2 Implicit Flow needs to be used instead. There is no UI to enable that: 
the AAD JSON manifest needs to be downloaded, updated, and uploaded by clicking on the Manage Manifest menu.

![AAD Manifest]({{ site.url }}/images/aad_manifest.png)

In the JSON file look for the `oauth2AllowImplicitFlow` property and set it to `true`.

# Project setup

Any IDE and web server can be used to write and run the code. I used Visual Studio 2015 which quickly creates a no-frills website using the 'File > Open > Web Site' menu
and starts an IIS Express web server when hitting F5. It also comes with Bower [pre-installed](http://docs.asp.net/en/latest/client-side/bower.html), which
simplifies package management. To get started open a command prompt and run:

{% highlight JavaScript %}
bower init
bower install <package> -save
{% endhighlight %}

where `<package>` is:

adal-angular | handles AAD auth in AngularJS modules
angular | Web App framework
angular-route | UI URL routing (required by ADAL Angular)
bootstrap | style
fontawesome | icons
require-css | loads CSS files
requirejs | module loader
requirejs-domready | delays running the RequireJS callback until the DOM is ready

# Web App

The full code is available in this [GitHub repo](https://github.com/maitre-matt/PowerBI-AngularJS) and a few points of interest are highlighted below.

The code relies on [ADAL.JS](https://github.com/AzureAD/azure-activedirectory-library-for-js) to retrieve access tokens from AAD and to attach them as HTTP headers (aka Bearer tokens) during REST calls.
ADAL.JS needs to be given the Tenant and Client IDs written down earlier. The Power BI REST endpoint also needs to be added and white-listed to enable authenticated CORS REST calls.

{% highlight JavaScript %}
adalProvider.init({
    tenant: "72f988bf-86f1-41af-91ab-2d7cd011db47", // microsoft.onmicrosoft.com
    clientId: "92f12926-5cf0-4460-83b6-14366bbaa88a", // Power BI AngularJS SPA
    endpoints: {
        "https://api.powerbi.com": "https://analysis.windows.net/powerbi/api",
    },
    requireADLogin: true,
    cacheLocation: 'localStorage'
    },
    $httpProvider
);

$sceDelegateProvider.resourceUrlWhitelist([
    'self',
    'https://*.powerbi.com/**'
]);
{% endhighlight %}

The Power BI REST APIs can then be called to get a list of reports, along with their associated embedding URLs:

{% highlight JavaScript %}
$http.get('https://api.powerbi.com/beta/myorg/reports')).then(function (response) {
    $scope.reports = response.data.value;
    $scope.selectedReport = $scope.reports[0].embedUrl;
});
{% endhighlight %}

An `iframe` receives those URLs via AngularJS scopes:

{% highlight HTML %}
<iframe id="report" ng-show="selectedReport" src="{ {selectedReport} }" frameborder="0" seamless style="height: 400px;" class="col-sm-12"></iframe>
{% endhighlight %}

and when the `iframe` is done loading the report, the AAD access token is sent to Power BI via an `iframe` message:

{% highlight JavaScript %}
var iframe = document.getElementById("report");
iframe.addEventListener("load", function () {
    var token = adal.getCachedToken("https://analysis.windows.net/powerbi/api");
    iframe.contentWindow.postMessage(JSON.stringify({ action: "loadReport", accessToken: token }), "*");
});
{% endhighlight %}
