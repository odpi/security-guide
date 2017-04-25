# Perimeter Security

Within the commonly deployed layered security or defense in depth approach to securing a system, the perimeter security is the outer most layer of infrastructure that is deployed to prevent unauthorized access to your Hadoop deployment.

Typical perimeter defenses include technologies like firewalls, intrusion detection systems \(IDS\), application proxies and virtual private network \(VPN\) servers. A combination of these technologies will provide a hardened exterior to your deployment. It should be noted that this is only one layer of the "onion" and that the deployment of distributed systems like Hadoop need to also protect access to compute, data and metadata resources within the cluster itself.

# Apache Knox

This section will discuss the use of Apache Knox as a type of application proxy within the perimeter layer. An application proxy receives requests intended for another server and acts on the client's behalf to obtain the requested resource. Application proxy servers are often used when the client and the server are incompatible for direct connection. For example, the client is unable to meet the security authentication requirements of the server but should be permitted some services. Characteristics of an application proxy include the following:

* Breaks the TCP/IP connection between a client and server; IP forwarding is not required.
* Hides the internal cluster IP addresses; only the IP address of the proxy server is visible to clients
* Provides detailed access logs
* Authenticates users and can assert the authenticated identity to the intended server
* Caches information

Knox satisfies nearly all of the characteristics of an application proxy as defined above. Currently, Knox does not do any caching other than some authentication events to minimize load on LDAP servers or other IDPs.

Apache Knox consists of three major sets of services that the user or developer will interact with:

1. Proxying Services
2. Authentication Services
3. Client Services

Each of the above sets of services plays a role in protecting the resources of Hadoop clusters or in interacting with the application proxy in order to securely access the resources through the gateway.

# Proxying Services of Apache Knox

The proxying services of Apache Knox represent the primary charter of the Knox component and allow the administrator to declaratively describe the Hadoop resources that may be proxied and the access requirements for securely gaining access to them. This is done through Knox Topologies. Knox Topologies are XML deployment descriptors that are generally used to represent:

* Unique Hadoop clusters - ie: dev vs prod
* Unique access requirements to the same cluster - ie: HTTP Basic vs token based or org specific LDAP searchbases
* The Admin API for Apache Knox is typically deployed in a specific topology called admin.xml but could be called anything.
* The KnoxSSO feature which is part of Authentication Services is also often in a specialized topology called knoxsso.xml but you may have others as well

**Listing 1. sandbox.xml an Example Topology File**

> &lt;topology&gt;

> &lt;gateway&gt;

> &lt;provider&gt;
>
>             &lt;role&gt;authentication&lt;/role&gt;
>
>             &lt;name&gt;ShiroProvider&lt;/name&gt;
>
>             &lt;enabled&gt;true&lt;/enabled&gt;
>
>             &lt;param&gt;

> &lt;name&gt;sessionTimeout&lt;/name&gt;
>
>                 &lt;value&gt;30&lt;/value&gt;
>
>             &lt;/param&gt;
>
>             &lt;param&gt;
>
>                 &lt;name&gt;main.ldapRealm&lt;/name&gt;
>
>                 &lt;value&gt;org.apache.hadoop.gateway.shirorealm.KnoxLdapRealm&lt;/value&gt;
>
>             &lt;/param&gt;
>
>             &lt;param&gt;
>
>                 &lt;name&gt;main.ldapContextFactory&lt;/name&gt;
>
>                 &lt;value&gt;org.apache.hadoop.gateway.shirorealm.KnoxLdapContextFactory&lt;/value&gt;
>
>             &lt;/param&gt;
>
>             &lt;param&gt;
>
>                 &lt;name&gt;main.ldapRealm.contextFactory&lt;/name&gt;
>
>                 &lt;value&gt;$ldapContextFactory&lt;/value&gt;
>
>             &lt;/param&gt;
>
>             &lt;param&gt;
>
>                 &lt;name&gt;main.ldapRealm.userDnTemplate&lt;/name&gt;
>
>                 &lt;value&gt;uid={0},ou=people,dc=hadoop,dc=apache,dc=org&lt;/value&gt;
>
>             &lt;/param&gt;
>
>             &lt;param&gt;
>
>                 &lt;name&gt;main.ldapRealm.contextFactory.url&lt;/name&gt;
>
>                 &lt;value&gt;ldap://localhost:33389&lt;/value&gt;
>
>             &lt;/param&gt;
>
>             &lt;param&gt;
>
>                 &lt;name&gt;main.ldapRealm.contextFactory.authenticationMechanism&lt;/name&gt;
>
>                 &lt;value&gt;simple&lt;/value&gt;
>
>             &lt;/param&gt;
>
>             &lt;param&gt;
>
>                 &lt;name&gt;urls./\*\*&lt;/name&gt;
>
>                 &lt;value&gt;authcBasic&lt;/value&gt;
>
>             &lt;/param&gt;
>
>         &lt;/provider&gt;

> &lt;provider&gt;
>
>             &lt;role&gt;identity-assertion&lt;/role&gt;
>
>             &lt;name&gt;Default&lt;/name&gt;
>
>             &lt;enabled&gt;true&lt;/enabled&gt;
>
>         &lt;/provider&gt;

> &lt;provider&gt;
>
>             &lt;role&gt;authorization&lt;/role&gt;
>
>             &lt;name&gt;AclsAuthz&lt;/name&gt;
>
>             &lt;enabled&gt;true&lt;/enabled&gt;
>
>         &lt;/provider&gt;
>
>     &lt;/gateway&gt;

> &lt;service&gt;
>
>         &lt;role&gt;NAMENODE&lt;/role&gt;
>
>         &lt;url&gt;hdfs://localhost:8020&lt;/url&gt;
>
>     &lt;/service&gt;

> &lt;service&gt;
>
>         &lt;role&gt;JOBTRACKER&lt;/role&gt;
>
>         &lt;url&gt;rpc://localhost:8050&lt;/url&gt;
>
>     &lt;/service&gt;

> &lt;service&gt;
>
>        &lt;role&gt;KNOXTOKEN&lt;/role&gt;
>
>        &lt;param&gt;
>
>           &lt;name&gt;knox.token.ttl&lt;/name&gt;
>
>           &lt;value&gt;300000000&lt;/value&gt;
>
>        &lt;/param&gt;
>
>        &lt;param&gt;
>
>           &lt;name&gt;knox.token.audiences&lt;/name&gt;
>
>           &lt;value&gt;tokenbased&lt;/value&gt;
>
>        &lt;/param&gt;
>
>        &lt;param&gt;
>
>           &lt;name&gt;knox.token.target.url&lt;/name&gt;
>
>           &lt;value&gt;https://localhost:8443/gateway/tokenbased&lt;/value&gt;
>
>        &lt;/param&gt;
>
>        &lt;param&gt;
>
>           &lt;name&gt;knox.token.client.data&lt;/name&gt;
>
>           &lt;value&gt;cookie.name=hadoop-jwt,test=value&lt;/value&gt;
>
>        &lt;/param&gt;
>
>     &lt;/service&gt;

> &lt;service&gt;
>
>         &lt;role&gt;WEBHDFS&lt;/role&gt;
>
>         &lt;url&gt;http://localhost:50070/webhdfs&lt;/url&gt;
>
>     &lt;/service&gt;

> &lt;service&gt;
>
>         &lt;role&gt;WEBHCAT&lt;/role&gt;
>
>         &lt;url&gt;http://localhost:50111/templeton&lt;/url&gt;
>
>     &lt;/service&gt;

> &lt;service&gt;
>
>         &lt;role&gt;OOZIE&lt;/role&gt;
>
>         &lt;url&gt;http://localhost:11000/oozie&lt;/url&gt;
>
>     &lt;/service&gt;

> &lt;service&gt;
>
>         &lt;role&gt;WEBHBASE&lt;/role&gt;
>
>         &lt;url&gt;http://localhost:60080&lt;/url&gt;
>
>     &lt;/service&gt;

> &lt;service&gt;
>
>         &lt;role&gt;HIVE&lt;/role&gt;
>
>         &lt;url&gt;http://localhost:10001/cliservice&lt;/url&gt;
>
>     &lt;/service&gt;

> &lt;service&gt;
>
>         &lt;role&gt;RESOURCEMANAGER&lt;/role&gt;
>
>         &lt;url&gt;http://localhost:8088/ws&lt;/url&gt;
>
>     &lt;/service&gt;
>
> &lt;/topology&gt;

The above listing illustrates the use of the authentication provider called the ShiroProvider. This is the default provider for Apache Knox and is generally used for HTTP Basic Authentication against an LDAP or Active Directory server within the deployment or enterprise. It is one of a number of out-of-the-box authentication and federation providers for Apache Knox. This particular configuration assumes the use of the DEMO LDAP server for Knox and is not intended to be used in production.

Another popular configuration for the ShiroProvider is to use PAM for authentication. Apache Knox documentation for the available providers may be found [here](http://knox.apache.org/books/knox-0-12-0/user-guide.html#Authentication).

As you can see in the listing above, the services that are to be exposed and proxied by the gateway are denoted by the &lt;service&gt; elements and indicate the URL that the gateway will dispatch requests to on behalf of the authenticated user.

You can also see that the &lt;gateway&gt; element and its child &lt;provider&gt; elements represent the access requirements for clients to access the proxied resources as well as the requirements for asserting the effective user identity to the backend services.

# Authentication Services of Apache Knox

A topology specific pipeline of providers has always been part of the proxying services in Apache Knox. The simplicity of composing a chain of Authentication/Federation, Identity Assertion and Authorization providers together actually grew into a separate set of services within Knox called Authentication Services. There are a number of ways that the authentication services are used:

1. For the Proxying Services, Apache Knox has needed to provide authentication mechanism flexibility for clients that cannot or would rather not use kerberos, for clients that don't want to use username and password and would rather use a token, etc. Authentication Services are an integral part of the Proxy Services provided by Knox.

2. KnoxSSO is a WebSSO flow for single-sign-on that leverages the authentication services to provide an sp-initiated SSO flow for the UIs in a Hadoop deployment. Applications may have explicit integrations with KnoxSSO or some may be able to leverage the combination of Knox proxying services and the SSOCookieProvider to gain SSO support.

3. KnoxToken Service is a REST API within the authentication services that returns a JWT token to represent an authentication event and provide a token based session until it expires or is terminated. Generally, this token is presented to the server as a Bearer token and is validated by a specific federation provider called the JWTProvider.

# Client Services of Apache Knox

The gateway-shell module of Apache Knox is the provider of Client Services for the project. This module includes two separate programming models for interacting with Apache Knox through the use of a Apache Groovy based DSL or by using the underlying classes directly as a Java SDK.

While typical REST/HTTP clients may be used to interact with Apache Knox, the client services provided by the gateway-shell module has some nice integrations built in for establishing token based sessions using the Authentication Services of Knox and powerful async capabilities within the DSL for writing scripts to be executed from the client desktop against Apache Knox within the perimeter.

