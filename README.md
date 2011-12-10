# Nginx Auth Request Module

## Introduction 

The `ngx_http_auth_request_module` is a module authored by
[Maxim Dounin](http://mdounin.ru), member of the core
[Nginx](http://nginx.org) team.

Maxim mantains a mercurial
[repository](http://mdounin.ru/hg/ngx_http_auth_request_module) with
the latest version of the code.

The module allows for the *insertion* of subrequests in the
authorization process being handled by Nginx. A more or less obvious
application is using this module as a very fast and specific
WAF. Meaning a WAF without all the ugliness that something like
Apache's `mod_security` promotes. No bizarre syntax, no reloading of
the config. Just add a small script that implements the set of
filtering rules that you want to implement. There's no need for any
behemoth module that hurts server performance and keep a consultant on
a retainer.

Suppose that you want to filter any occurence of the `UNION`, `SELECT`
and `DROP` statement in the URI. Suppose that the backoffice of your
app is located at `/bo`. This will do the trick:

    location ^~ /bo {
        auth_basic 'Classified Access';
        auth_basic_user_file .htpasswd-bo-users;
        auth_request /auth;
    }

    location /auth {
        if ($request_uri ~* (?:select|union|drop)) {
           return 403;
        }
        return 200;
    }
    
Let's follow the example. There's a request for an URI like this:

    /bo/?u=john_doe'; DROP TABLE users--
    
 1. This request is served by the `^~ /bo` location. There the
    `request_auth` location redirects to `/auth`.
    
 2. Now we're on `/auth` and we check the `if` condition regex. We
    have a match with `DROP`. Hence we deny authorization by returning
    a 403 status code, i.e., `403 Access Denied`.
    
 3. If we had requested an **acceptable** URI then we returned 200
    from the `/auth` location and we proceeded as usual to the
    following stage.
    
You might say that this example didn't needed to use `auth_request` in
the first place. Indeed you could have done everything using a simple
[`map`](http://wiki.nginx.org/HttpMapModule#map) directive like this:

    map $request_uri $attempt_sql_injection {
        default 0;
        ~*(?:select|union|drop) 1;
    }

include it in the `http` context of your Nginx instance and set:
    
    location ^~ /bo {
        auth_basic 'Classified Access';
        auth_basic_user_file .htpasswd-bo-users;
        
        if ($attempt_sql_injection) {
            return 403;
        }
    }
    
Yes. But this was just an example to give you a hint towards
realizing all the potential uses of this handy module.
    
Here's a quick braindump of potential uses of the `auth_request` module.
    
 1. Easily implement [two-factor authentication](https://en.wikipedia.org/wiki/Two-factor_authentication)
    by invoking a script in the `/auth` location.
    
 2. Implement Single Sign On systems.
 
 3. Filter requests and prevent vulnerabilities like
    [SQL injection](https://www.owasp.org/index.php/SQL_injection),
    [XSS](https://www.owasp.org/index.php/Cross-site_Scripting_%28XSS%29),
    etc.
    
 4. Complex caching strategies.
 
 5. Web services implementations. 
 
And much, much more. No limit to your imagination. Dream on.

## Module directives

**auth_request** `uri` | `off`

**default:** `off`

**context:** `http`, `server`, `location`

Enables the `auth_request` module in a given context where the request
for authorization will be made to the `uri` defined.

<br/>
<br/>

**auth_request_set** `variable` `value`

**default:** `none`

**context:** `http`, `server`, `location`

Sets the value of the `variable` to `value` after the completion of
the authorization request.

## Caveats 

Note that the module discards the request body. Therefore if you're
proxying to an upstream you must set
[`proxy_pass_request_body`](http://wiki.nginx.org/HttpProxyModule#proxy_pass_request_body)
to `off` and set the `Content-Length` header to a null string. Here's
and example:

    location = /auth {
        proxy_pass http://auth_nackend;
        proxy_pass_request_body off;
        proxy_set_header Content-Length "";
        proxy_set_header X-Original-URI $request_uri;
    }
    
You cannot use
[`proxy_cache`](http://wiki.nginx.org/HttpProxyModule#proxy_cache)/[`proxy_store`](http://wiki.nginx.org/HttpProxyModule#proxy_store)
or
[`fastcgi_cache`](http://wiki.nginx.org/HttpFcgiModule#fastcgi_cache)/[`fastcgi_store`](http://wiki.nginx.org/HttpFcgiModule#fastcgi_store)
for requests involving `auth_request`.

## Installation 

 1. Grab the source from the mercurial
    [tip](http://mdounin.ru/hg/ngx_http_auth_request_module/archive/tip.tar.gz).
    
 2.  2. Add the module to the build configuration by adding
    `--add-module=/path/to/ngx_http_auth_request_module`.
    
 3. Build the nginx binary.
 
 4. Install nginx binary.
 
 5. Configure contexts where `auth_request` is enabled.
 
 6. Done.
 
## Other builds
 
If you fancy a bleeding edge Nginx package (from the dev releases) for
Debian made to measure then you might be interested in my
[debian](http://debian.perusio.net/unstable) Nginx
package. Instructions for using the repository and making the package
live happily inside a stable distribution installation are
[provided](http://debian.perusio.net).

## Acknowledgments

Thanks to [Maxim Dounin](http://mdounin.ru) for making the module
available and being so helpful on the
[Nginx mailing list](nginx.org/mailman/listinfo/nginx) answering all
kinds of issues and particularly regarding this module.
