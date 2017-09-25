---
layout: post
title: "ASP.NET Quick and Easy Security Fixes"
subtitle: "Easy fixes for low severity or informational pentest issues"
tags: [dotnet, security]
---

This is more of a personal reminder of things I need to do on each project I start to keep down the number of small annoying issues raised as part of a penetration test.


## Hide Application and Framework version details

This is a nice, easy quick win

_Web.config_
```xml
<configuration>
    <system.web>
        <httpRuntime enableVersionHeader="false" />
    </system.web>
    <system.webServer>
        <httpProtocol>
            <customHeaders>
                <remove name="X-Powered-By" />
                <remove name="X-Aspnet-Version" />
            </customHeaders>
        </httpProtocol>
    </system.webServer>
</configuration>
```

## Prevent Clickjacking

The act of someone embedding your web app within an iFrame and capturing click events to do potentially harmful things. More info on [wikipedia](https://en.wikipedia.org/wiki/Clickjacking).

Simplest way is to prevent the application from being loaded into an iFrame at all, or at least limit the locations where it can be.

_Web.config_
```xml
<configuration>
    <system.webServer>
        <httpProtocol>
            <customHeaders>
                <!-- 

                DENY               Disallow embedding.
                SAMEORIGIN         Allow embedding of own content only.
                ALLOW-FROM origin  Allow specific origins to embed this content 

                -->
                <add name="X-Frame-Options" value="DENY" />

                <!-- 
                
                frame-ancestors valid values.

                none               Disallow embedding.
                self               Allow embedding of own content only.
                a.com b.co.uk      Allow specific origins to embed this content 
                
                -->
                <add name="Content-Security-Policy" value="frame-ancestors 'none'">
            </customHeaders>
        </httpProtocol>
    </system.webServer>
</configuration>
```

You could also implement this as an IHttpModule:

_XFrameOptionsHeader.cs_
```csharp
public class XFrameOptionsHeader : IHttpModule
{
    public void Init(HttpApplication context)
    {
        context.PreSendRequestHeaders += OnPreSendRequestHeaders;
    }

    public void Dispose() { }

    void OnPreSendRequestHeaders(object sender, EventArgs e)
    {
        var app = sender as HttpApplication;

        if (app != null && app.Context != null)
        {
            var headers = app.Response.Headers;
            if (headers["X-Frame-Options"] == null)
                headers.Add("X-Frame-Options", "DENY");
            else
                headers["X-Frame-Options"] = "DENY";

            if (headers["Content-Security-Policy"] == null)
                headers.Add("Content-Security-Policy", "frame ancestors 'none'");
            else
                headers["Content-Security-Policy"] = "frame ancestors 'none'";
        }
    }
}
```


_Web.config_
```xml
<configuration>
    <system.webServer>
        <modules>
            <!-- 'type' will need to be qualified with the namespace in which the module exists -->
            <add name="XFrameOptionsHeader" type="XFrameOptionsHeader" />
        </modules>
    </system.webServer>
</configuration>
```

Content-Security-Policy supercedes X-Frame-Options and browsers that understand it will use it instead, from my understanding there is no harm in including both to provide a greater browser compatibility.

## Disable caching

This is probably a bit on the extreme side and can be tailored to the specific application but to prevent caching of pages that may contain personal data.

_Web.config_
```xml
<configuration>
    <system.webServer>
        <httpProtocol>
            <customHeaders>
                <add name="Pragma" value="no-cache" />
                <add name="Cache-Control" value="no-store" />
            </customHeaders>
        </httpProtocol>
    </system.webServer>
</configuration>
```

## Hide IIS Version

The IIS Version is exposed by default on every request in the **Server** HTTP Response header, it is added directly by IIS and cannot be removed using web.config customHeaders. The method I use is to create a httpModule to remove the header

_RemoveServerVersionHeader.cs_
```csharp
public class RemoveServerVersionHeader : IHttpModule
    {
        public void Init(HttpApplication context)
        {
            context.PreSendRequestHeaders += OnPreSendRequestHeaders;
        }

        public void Dispose() { }

        void OnPreSendRequestHeaders(object sender, EventArgs e)
        {
            var app = sender as HttpApplication;

            if(app != null && app.Context != null)
                app.Response.Headers.Remove("Server");
        }
    }
```

_Web.config_
```xml
<configuration>
    <system.webServer>
        <modules>
            <!-- 'type' will need to be qualified with the namespace in which the module exists -->
            <add name="RemoveServerVersionHeader" type="RemoveServerVersionHeader" />
        </modules>
    </system.webServer>
</configuration>
```

## HTTP Strict Transport Security

Sending this header in the response will prevent the browser from requesting communication over HTTP on the current domain, everything will be forced to HTTPS.

However, we only want to add this if the original request comes over HTTPS and the ```IsSecure``` method checks the standard ```HttpContext.Request.IsSecureConnection``` property. If your application lives behind a load balancer with SSL offloading you will only see plain HTTP requests coming in- in this case they will normally add in a custom header which we look for as required.

_HttpStrictTransportSecurity.cs_
```csharp
public class HttpStrictTransportSecurity : IHttpModule
{
    public void Init(HttpApplication context)
    {
        context.PreSendRequestHeaders += OnPreSendRequestHeaders;
    }

    public void Dispose() { }

    void OnPreSendRequestHeaders(object sender, EventArgs e)
    {
        var context = GetContext(sender);
        if (context != null && IsSecure(context))
            context.Response.Headers.Add("Strict-Transport-Security", "max-age=31536000");
    }

    HttpContext GetContext(object sender)
    {
        var app = sender as HttpApplication;
        if (app != null && app.Context != null)
            return app.Context;

        return null;
    }

    static bool IsSecure(HttpContext context)
    {
        // custom headers that indicate that even though the request is over http, the site is behind
        // a secure firewall or load balancer with ssl offloading.

        // Microsoft (TMG etc)
        var frontEndHttps = context.Request.Headers["Front-End-Https"] ?? "off";

        // Others
        var forwardedProto = context.Request.Headers["X-Forwarded-Proto"] ?? "http";

        // Return true for native secure requests or where the headers above are the correct value
        return context.Request.IsSecureConnection ||
                frontEndHttps.Equals("on", StringComparison.InvariantCultureIgnoreCase) ||
                forwardedProto.Equals("https", StringComparison.InvariantCultureIgnoreCase);
    }
}
```

_Web.config_
```xml
<configuration>
    <system.webServer>
        <modules>
            <!-- 'type' will need to be qualified with the namespace in which the module exists -->
            <add name="HttpStrictTransportSecurity" type="HttpStrictTransportSecurity" />
        </modules>
    </system.webServer>
</configuration>
```

## Mmmm Cookies!

Finally we set httpOnly flag on any cookies we write to tell the browser that these cookies should not be accessible to client side script, only the web server. Optionally, we can also flag them as SSL only and the browser should not send them if we make any plain HTTP requests to the server by mistake or otherwise.

_Web.config_
```xml
<configuration>
    <system.web>
        <httpCookies httpOnlyCookies="true" requireSSL="true" />
    </system.web>
</configuration>
```

Of course there are other thing to do to ensure a web application is not vulnerable to malicious users, but this is the basic _configuration_ fixes that every project needs.