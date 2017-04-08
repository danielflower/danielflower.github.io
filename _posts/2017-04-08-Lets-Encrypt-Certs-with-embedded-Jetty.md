---
layout: post_page
title: LetsEncrypt certs with embedded Jetty on 
---

This blog is for you if you use embedded Jetty on linux (including Amazon's own Linux variation on EC2) and want free
SSL certs that automatically renew themselves. This describes what I did to make certificates work for
[https://www.crickam.com/](https://www.crickam.com/)

It covers:
* Generating a LetsEncrypt cert and converting it to a keystore that Jetty can use
* Adding a script to automatically renew the cert, which runs daily (it only renews the cert if it is near expiry)
* Getting Jetty to automatically refresh the renewed cert, without having to restart the server and with no downtime.

This blog will go over the scripts I made and the changes I made to my Jetty app. I'll assume you've managed to
install and can run `~/certbot-auto`

Enabling certbot to validate certificates
-----------------------------------------

When creating certificates, you need to put able to prove that you own the domain name. `cerbot` does this for you
automatically by hosting a file on your domain. While certbot can run in "standalone" mode (where it temporarily binds
to your server's port to serve the file itself), if you want to generate certs while your app is running you need to be
able to serve files yourself from the file system. The app I was using was an uber jar serving everything from the classpath,
so I added a handler especially for certbot to use for cert validations:

````java
private static ResourceHandler dirForLetsEncryptValidation(String dir) {
    ResourceHandler resourceHandler = new ResourceHandler();
    resourceHandler.setResourceBase(dir);
    return resourceHandler;
}
    
public void createServer() {
    // ...snip...
    handlers.addHandler(dirForLetsEncryptValidation("/opt/myapp/letsencryptvalidationdir");
    // ...snip...
}
````

Generating the cert for the first time
--------------------------------------

This is one off, so just ran manually from my user's home dir: `sudo ./certbot-auto certonly`

It will ask some questions: select `Place files in webroot directory (webroot)` and give it the path to the directory
specified in the special resource handler - `/opt/myapp/letsencryptvalidationdir` in my example. Assuming your web app
is running (it will use http on port 80, or https on port 443, not sure which) then it will output a bunch of files
in a directory similar to `/etc/letsencrypt/live/www.your-domain.com`

These are pem files - to convert this to a keystore format that Java could use, I created the following script (which
needs to run as root), which lives at `~/convertkeystore.sh`

````bash
#!/bin/sh
set -e
set -x

(
  cd /etc/letsencrypt/live/www.crickam.com
  openssl pkcs12 -export -in fullchain.pem -inkey privkey.pem -out /opt/crickam/keystore.p12 -name crickam -CAfile chain.pem -caname root -password file:/opt/crickam/keystore.pw
  chmod a+r /opt/crickam/keystore.p12
)
````

Where:
* `-out` is the path to the keystore you want to create
* `-name` can be whatever you want (I'm not sure what it's used for)
* `-password file:path` is the path to a file that just contains the keystore password (my java code reads this same 
file to access the keystore password)
* All other options should be left as is, as they point to files that certbot created

Using the generated key
-----------------------

It's quite straightforward - just create an `SslContextFactory` and point to the generated `keystore.p12` file:

````java
Server jettyServer = new Server();
HttpConfiguration config = new HttpConfiguration();
config.addCustomizer(new SecureRequestCustomizer());
config.addCustomizer(new ForwardedRequestCustomizer());

// Create the HTTP connection
HttpConnectionFactory httpConnectionFactory = new HttpConnectionFactory(config);
ServerConnector httpConnector = new ServerConnector(jettyServer, httpConnectionFactory);
httpConnector.setPort(8080); // IP tables redirect 80 -> 8080
jettyServer.addConnector(httpConnector);

// Create the HTTPS end point
SslContextFactory sslContextFactory = new SslContextFactory();
sslContextFactory.setKeyStoreType("PKCS12");
sslContextFactory.setKeyStorePath("/opt/crickam/keystore.p12");
String keyStorePassword = FileUtils.readFileToString(new File("/opt/crickam/keystore.pw"), "UTF-8").trim();
sslContextFactory.setKeyStorePassword(keyStorePassword);
sslContextFactory.setKeyManagerPassword(keyStorePassword);
ServerConnector httpsConnector = new ServerConnector(jettyServer, sslContextFactory, new HttpConnectionFactory(config));
httpsConnector.setPort(8443); // IP tables redirect 8443 -> 443
jettyServer.addConnector(httpsConnector);
````

With this, you should be able to use your certificate, however it will expire in 90 days. So time to automate some stuff.

Automated renewal and hot reloading of SSL certs
------------------------------------------------

After the first manual creation of the cert, certbot will remember the cert and so renewel of the cert is extremely easy:
just run `sudo ./certbot-auto renew`

This will check the validity of the cert and do nothing if it is okay, or renew it if it has been revoked or near expiry.
If it is renewed, we want to convert the cert to a java keystore. With certbot, you can specify a hook that only runs if
it actually renews the cert, so we can tell it to run the script specified above if it needs to:

    sudo ./certbot-auto renew --post-hook "./convertkeystore.sh"

So that's nice, but Jetty will still be using the old certificate for requests. Fortunately, on newer versions of Jetty
you can tell it to hot reload your certs by calling the following on the `SslContextFactory` you create:

````java
sslContextFactory.reload(scf -> log.info("Reloading cert"));
````

Now the problem is to know when to actually run this code. What I did was expose a diagnostic URL that runs the code
(but only if you know the secret password):

````java
import org.eclipse.jetty.server.Request;
import org.eclipse.jetty.server.handler.AbstractHandler;
import org.eclipse.jetty.util.ssl.SslContextFactory;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

public class DiagnosticsHandler extends AbstractHandler {
    private static final Logger log = LoggerFactory.getLogger(DiagnosticsHandler.class);
    private final SslContextFactory sslContextFactory;

    public DiagnosticsHandler(SslContextFactory sslContextFactory) {
        this.sslContextFactory = sslContextFactory;
    }

    @Override
    public void handle(String target, Request baseRequest, HttpServletRequest request, HttpServletResponse response)
                       throws IOException, ServletException {

        if (target.equals("/diagnostics/refresh-certs") && request.getMethod().equals("PUT")) {
            if ("some-secret-password".equals(request.getParameter("password"))) {
                try {
                    sslContextFactory.reload(sslContextFactory -> log.info("Reloading cert"));
                    response.setStatus(200);
                    response.getWriter().append("Reloaded").close();
                } catch (Exception e) {
                    log.error("Error reloading cert", e);
                    response.sendError(500, "Error while reloading cert: " + e.getMessage());
                }
            } else {
                log.warn("Invalid password");
                response.sendError(403, "Incorrect password");
            }
            baseRequest.setHandled(true);
        }

    }
}
````

The `SslContextFactory` is the one created above, so this class is simply `new`'ed up with that and added to Jetty's
handlers:

````java
handlers.addHandler(new DiagnosticsHandler(sslContextFactory));
````

Now you can tell Jetty to reload SSL certs using cURL:

    curl --insecure -X PUT "https://localhost:8443/diagnostics/refresh-certs?password=some-secret-password"

Note: the localhost URL uses port 8443, but externally it would use the default port (443), thanks again to iptables.

Putting it all together and scheduling the updates
--------------------------------------------------

The following script puts together the certbot check, the conversation of the keystore, and the cURL to tell jetty
to reload the certs, and lives in `/home/ec2-user/renewcerts.sh`

````bash
#!/bin/sh
set -e
set -x

/home/ec2-user/certbot-auto renew --post-hook "/home/ec2-user/convertkeystore.sh && curl --insecure -X PUT \"https://localhost:8443/diagnostics/refresh-certs?password=some-secret-password\" --silent"
````

It needs to run as root, so `sudo crontab -e` to schedule it:

    39 14 * * * /home/ec2-user/renewcerts.sh > /opt/crickam/logs/cronoutput.log 2>&1

That runs at 2:39pm every day - doesn't matter when exactly, and it could run hourly, daily, or weekly, as normally it
will do nothing.

Done
----

That's it. You now have free SSL certs that will auto renew themselves.