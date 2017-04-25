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

`<?xml version="1.0" encoding="utf-8"?>`

`<topology>`

`    <gateway>`

`        <provider>`

`            <role>authentication</role>`

`            <name>ShiroProvider</name>`

`            <enabled>true</enabled>`

`            <param>`

`                <name>sessionTimeout</name>`

`                <value>30</value>`

`            </param>`

`            <param>`

`                <name>main.ldapRealm</name>`

`                <value>org.apache.hadoop.gateway.shirorealm.KnoxLdapRealm</value>`

`            </param>`

`            <param>`

`                <name>main.ldapContextFactory</name>`

`                <value>org.apache.hadoop.gateway.shirorealm.KnoxLdapContextFactory</value>`

`            </param>`

`            <param>`

`                <name>main.ldapRealm.contextFactory</name>`

`                <value>$ldapContextFactory</value>`

`            </param>`

`            <param>`

`                <name>main.ldapRealm.userDnTemplate</name>`

`                <value>uid={0},ou=people,dc=hadoop,dc=apache,dc=org</value>`

`            </param>`

`            <param>`

`                <name>main.ldapRealm.contextFactory.url</name>`

`                <value>ldap://localhost:33389</value>`

`            </param>`

`            <param>`

`                <name>main.ldapRealm.contextFactory.authenticationMechanism</name>`

`                <value>simple</value>`

`            </param>`

`            <param>`

`                <name>urls./**</name>`

`                <value>authcBasic</value>`

`            </param>`

`        </provider>`

`        <provider>`

`            <role>identity-assertion</role>`

`            <name>Default</name>`

`            <enabled>true</enabled>`

`        </provider>`

`    </gateway>`

`    <service>`

`        <role>NAMENODE</role>`

`        <url>hdfs://localhost:8020</url>`

`    </service>`

`    <service>`

`        <role>JOBTRACKER</role>`

`        <url>rpc://localhost:8050</url>`

`    </service>`

`    <service>`

`       <role>KNOXTOKEN</role>`

`       <param>`

`          <name>knox.token.ttl</name>`

`          <value>300000000</value>`

`       </param>`

`       <param>`

`          <name>knox.token.audiences</name>`

`          <value>tokenbased</value>`

`       </param>`

`       <param>`

`          <name>knox.token.target.url</name>`

`          <value>https://localhost:8443/gateway/tokenbased</value>`

`       </param>`

`       <param>`

`          <name>knox.token.client.data</name>`

`          <value>cookie.name=hadoop-jwt,test=value</value>`

`       </param>`

`    </service>`

`    <service>`

`        <role>WEBHDFS</role>`

`        <url>http://localhost:50070/webhdfs</url>`

`    </service>`

`    <service>`

`        <role>WEBHCAT</role>`

`        <url>http://localhost:50111/templeton</url>`

`    </service>`

`    <service>`

`        <role>OOZIE</role>`

`        <url>http://localhost:11000/oozie</url>`

`    </service>`

`    <service>`

`        <role>WEBHBASE</role>`

`        <url>http://localhost:60080</url>`

`    </service>`

`    <service>`

`        <role>HIVE</role>`

`        <url>http://localhost:10001/cliservice</url>`

`    </service>`

`    <service>`

`        <role>RESOURCEMANAGER</role>`

`        <url>http://localhost:8088/ws</url>`

`    </service>`

`</topology>`

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

