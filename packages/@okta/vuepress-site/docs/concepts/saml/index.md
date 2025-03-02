---
title: Understanding SAML
layout: docs_page
icon: /assets/img/icons/saml.svg
meta:
  - name: description
    content: Secure Authentication Markup Language is a standards-based protocol for exchanging digital authentication signatures. Learn how SAML operates and how to set up SAML applications in Okta.
---

# SAML

Traditionally, enterprise applications are deployed and run within the company network. To obtain information about users such as user profile and group information, many of these applications are built to integrate with corporate directories such as Microsoft Active Directory. More importantly, a user's credentials are typically stored and validated using the directory. For example, if you use SharePoint and Exchange that are running on premises, your login credentials are your Active Directory credentials.

However, with increased collaboration and the move towards the cloud, many applications have moved beyond the boundaries of a company's domain. Federated Authentication is the solution to this problem.

## Authentication

Before diving into federated authentication, we need to understand what authentication really means. Authentication defines the way a user is identified and validated through some sort of credentials as part of a login flow. Most applications will present a login page to an end user, allowing him to specify a username and a password. In some cases, additional information may be required in order to locate the user - like a company ID or a client code. This information allows the application to narrow down the search of the username applicable to the provided info. This is often used to allow the same username to exist across multiple tenants belonging to different customers.

Most applications will have a user store (DB or LDAP) containing, among other things, user profile information and credentials. During login, the credentials are validated against this backend user store. The advantage of this approach is that it is simple, because everything is managed within the application, providing a single and consistent way to authenticate an end user. However, if a user needs to access multiple applications where each app requires a different set of credentials, it becomes a hassle for the end user. First, the user will need to remember different passwords - in addition to any other corporate password (eg. AD password) that may already exist. The user is now forced to maintain separate usernames and passwords, dealing with different password policies and expirations. Second, this also creates a headache for administrators and ISVs when application users continue to have access to applications that should have been revoked.

## Federated Identity

Federated Identity started with the need to support application access that spans beyond a company or organization boundary. Imagine a relationship between a juice company (JuiceCo) selling its product to a large supermarket chain (BigMart). As an employee of JuiceCo, you need to access an application provided by BigMart to manage the relationship and monitor supplies and sales.

In this case, BigMart (who is providing this application) will need to take care of user authentication. The simple way is to require a different user name and password from users working at JuiceCo. But think about all the users that this application will need to maintain - including all of the other suppliers and their users who need to access the application.

A more elegant way to solve this problem is to allow JuiceCo and every other supplier to share or "federate" the identities with BigMart. As an employee of JuiceCo, you already have a corporate identity and credentials. What Federated Identity provides is a secure way for the supermarket chain (Service Provider) to externalize authentication by integrating with its suppliers' existing identity infrastructure (Identity Provider).

This type of use case is what led to the birth of federated protocols such as [Secure Assertion Markup Languange (SAML)](http://en.wikipedia.org/wiki/Security_Assertion_Markup_Language).

See [Security Assertion Markup Language (SAML) V2.0 Technical Overview](http://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html) for a in-depth overview.

## Planning for SAML

SAML is mostly used as a web-based authentication mechanism inasmuch as it relies on using the browser agent to broker the authentication flow. At a high-level, the authentication flow of SAML looks like this:

![SAML Flow](/img/saml_guidance_saml_flow.png "SAML Flow")

We are now ready to introduce some common SAML terms. We will go into the technical details of these later, but it is important to understand the high-level concept during the planning stage.

- **A Service Provider (SP)** is the entity providing the service, typically in the form of an application

- **An Identity Provider (IDP)** is the entity providing the identities, including the ability to authenticate a user. The Identity Provider typically also contains the user profile: additional information about the user such as first name, last name, job code, phone number, address, etc. Depending on the application, some service providers may require a very simple profile (username, email), while others may require a richer set of user data (job code, department, address, location, manager, etc).

- **A SAML Request**, also known as an authentication request, is generated by the Service Provider to "request" an authentication.

- **A SAML Response** is generated by the Identity Provider. It contains the actual assertion of the authenticated user. In addition, a SAML Response may contain additional information, such as user profile information and group/role information, depending on what the Service Provider can support.

- **A Service Provider Initiated (SP-initiated)** login describes the SAML login flow when initiated by the Service Provider. This is typically triggered when the end user tries to access a resource or login directly on the Service Provider side, such as when the browser tries to access a protected resource on the Service Provider side.

- **An Identity Provider Initiated (IDP-initiated)** login describes the SAML login flow initiated by the Identity Provider. Instead of the SAML flow being triggered by a redirection from the Service Provider, in this flow the Identity Provider initiates a SAML Response that is redirected to the Service Provider to assert the user's identity.

A couple of key things to note:

1. The Service Provider never directly interacts with the Identity Provider. A browser acts as the agent to carry out all the redirections.

2. The Service Provider needs to know which Identity Provider to redirect to before it has any idea who the user is.

3. The Service Provider does not know who the user is until the SAML assertion comes back from the Identity Provider.

4. This flow does not have to start from the Service Provider. An Identity Provider can initiate an authentication flow.

5. The SAML authentication flow is asynchronous. The Service Provider does not know if the Identity Provider will ever complete the entire flow. Because of this, the Service Provider does not maintain any state of any authentication requests generated. When the Service Provider receives a response from an Identity Provider, the response must contain all the necessary information.

## Planning Checklist

While the SAML protocol is a standard, there are different ways to implement it depending on the nature of your application. The following is a checklist that will guide you through some of key considerations.

  1. Understanding the role of a Service Provider

  2. Single IDP vs Multiple IDPs

  3. Understanding SP-initiated Login Flow

  4. Exposing SAML configuration in SP

  5. Enabling SAML for everyone vs a subset of users

  6. Implementing a "backdoor"

### Understanding the Role of a Service Provider

A SAML IDP generates a SAML response based on configuration that is mutually agreed upon by the IDP and the SP. Upon receiving the SAML assertion, the SP needs to validate that the assertion comes from a valid IDP and then parse the necessary information from the assertion: the username, attributes, etc.

In order to do this, the SP requires at least the following:

- Certificate - The SP needs to obtain the public certificate from the IDP to validate the signature. The certificate is stored on the SP side and used whenever a SAML response arrives.
- ACS Endpoint - Assertion Consumer Service URL - often referred to simply as the SP login URL. This is the endpoint provided by the SP where SAML responses are posted. The SP needs to provide this information to the IDP.
- IDP Login URL - This is the endpoint on the IDP side where SAML requests are posted. The SP needs to obtain this information from the IDP.

The easiest way to implement SAML is to leverage an OpenSource SAML toolkit. We have included a list at the end of this article of recommended toolkits for several languages. These toolkits provide the logic needed to digest the information in an incoming SAML Response. In addition, if the SP needs to support the SP-initiated Login flow, the toolkits also provide the logic needed to generate an appropriate SAML Authentication Request.

### Single IDP vs Multiple IDPs

If you are building an internal app and you want to SAML-enable it in order to integrate with your corporate SAML identity provider, then you are looking at supporting only a single IDP. In this case, your app only needs to deal with a single set of IDP metadata (cert, endpoints, etc).

![Single IDP](/img/saml_guidance_one_idp.png "Single IDP")

If you are an ISV building an enterprise SaaS product, or if you are building an external facing website/portal/community for your customers and partners, then you need to look at supporting multiple IDPs. This is the typical use case for many SaaS ISVs needing to integrate with customers' corporate identity infrastructure. Depending on the architecture of your application, you need to think about ways to store the SAML configuration (eg. Certificates, IDP login URLs) from each identity provider, as well as how you will provide the necessary SP information for each.

![Many IDPs](/img/saml_guidance_many_idp.png "Many IDPs")

A key consideration involves the ACSurl endpoint on the SP side where SAML responses are posted. It is possible to expose a single endpoint even when dealing with multiple IDPs. For a single-instance multi-tenant application where the tenancy is not defined in the URL (such as via a subdomain), this might be a simpler way to implement. However, you must then rely on additional information in the SAML response to determine which IDP is trying to authenticate (for example, using the IssuerID). If your application is architected in a multi-tenant fashion with domain information in the URL (for example, *https://domain1.myISV.com* or *https://www.myISV.com/domain1*), then having an ACSurl endpoint for each subdomain might be a good option since the URL itself identifies the domain.

![SPs with Subdomains](/img/saml_guidance_many_idp_subdomain.png "SPs with Subdomains")

### Understanding SP-initiated Login Flow

As discussed earlier, an IDP-initiated login flow starts from the IDP. Since it begins on the IDP side, there is no additional context about what the user is trying to access on the SP side other than the fact that the user is trying to get authenticated and access the SP. Typically, once the user is authenticated, the browser will be taken to a generic landing page in the SP.

In an SP-initiated flow, the user tries to access a protected resource directly on the SP side without the IDP being aware of the attempt. Two issues arise. First is the need to identify the right IDP if authentication of a federated identity is needed. With SP-initiated login, the SP initially does not know anything about the identity. As a developer, you need to figure out how the SP can determine which IDP should be receiving the SAML request. In some cases, if your application URLs contain subdomain information that is mapped to a unique tenant and IDP, then the resource link being hit is enough to identify the IDP. If this isn't the case, then you might need to prompt the end user for additional information from the end user such as user id, email, or a company id; something that allows the SP to identify which IDP the user attempting to access the resource belongs to. Remember, you are only prompting for an identifier, not credentials. Okta also supports to pass the identifier to the IDP with parameter "LoginHint", so that the user does not need to input the identifier again when redirected to IDP to login. For instruction to trigger Okta to send the "LoginHint" to IDP, see [Redirecting with SAML Deep Links](/docs/reference/api/idps#redirecting-with-saml-deep-links).

Another issue with SP-initiated login flow is the support for deep links. Most applications support deep links. For example, you might receive a link to a document that resides on a content management system. Ideally, if you need to authenticate prior to accessing the document, you would like to be taken to the document immediately after authentication.

SAML is an asynchronous protocol by design. The SP-initiated login begins the flow by generating a SAML Authentication Request that gets redirected to the IDP. At this point, the SP does not store any information about the request. When the SAML response comes back from the IDP, the SP wouldn't know anything about the initial deep-link that triggered the authentication request. Luckily, SAML supports this with a parameter called RelayState.

A RelayState is an HTTP parameter that can be included as part of the SAML request and SAML response. In an SP-initiated login flow, the SP can set the RelayState parameter in the SAML request with additional information about the request. A SAML IDP, upon receiving the SAML request, will take the RelayState value and simply attach it back as a HTTP parameter in the SAML response after the user has been authenticated. This way, when the round trip completes, the SP can use the RelayState information to get additional context about the initial SAML authentication request.

In the case of a deep link, the SP sets the RelayState of the SAML request with the deep-link value. When the SAML response comes back, the SP can use the RelayState value and take the authenticated user to the right resource.

![SP-initiated flow with Deep Link](/img/saml_guidance_deeplink.png "SP-initiated Login with Deep Link")

For instructions to construct a deep link for SAML IdPs, see [Redirecting with SAML Deep Links](/docs/reference/api/idps#redirecting-with-saml-deep-links).

### Exposing SAML configuration in SP

As discussed before, the SP needs the IDP configuration in order to complete the SAML setup. While many ISVs choose to do this via support and email, the better way to do this is by exposing a self-service administrator page for your customer's IT administrator to enable SAML. SAML supports metadata on both the IDP and SP side. One way to configure the IDP/SP relationship on the SP side is to build the ability to receive an IDP metadata file and the ability to generate an SP metadata file for consumption by the IDP. This is the preferred method.

However, some ISVs choose to allow configuration of several key SAML parameters directly rather than through a metadata file. Typical parameters would include the IDP redirect URL (for SAML Request), IssuerID, IDP Logout URL. The SP must also allow the IDP public certificate to be uploaded or saved.

Using a metadata file is preferred because it can handle any future additions/enhancements in your SAML support without making UI changes that would otherwise be required if you expose specific SAML configuration parameters in your UI.

### Enabling SAML for everyone vs a subset of users

Depending on the nature of your application, there might be reasons to allow only a subset of users to be SAML enabled. Imagine an application that is accessed by internal employees and external users like partners. The employees may SAML into the app, while the external users may use a separate set of credentials.
Even in cases where the intent is to have all the users of a particular tenant be SAML-enabled, it might be useful to enable just a subset of users during proof-of-concept, testing and roll-out to test out authentication with a smaller subset of users before going-live for the entire population.

### Implementing a "backdoor"

This is particularly important where the entire population is intended to be SAML-enabled in your application. Sometimes, there might be a mistake in the SAML configuration - or something changes in SAML IDP endpoints. In any case, you do not want to be completely locked out. Having a backdoor available for administrator(s) to use to access a locked system becomes extremely important. This is often accomplished by having a "secret" login URL that does not trigger a SAML redirection when accessed. Typically, the administrator will use a username/password to login in and make the necessary changes to fix the problem.

## Testing SAML

Use the [Okta SAML validation tool](http://saml.oktadev.com/) to speed up the process of developing a SAML SP.

This tool makes it easy for you to send SAML Requests to your SAML SP. It allows you to quickly change the contents of the SAML requests and simplifies the process of debugging SAML issues by automatically decoding SAML payloads and displaying server headers for you.

You can also install the [SAML Tracer extension to Firefox](https://addons.mozilla.org/en-US/firefox/addon/saml-tracer/) for testing, or similar tools for other browsers.

## SAML Toolkits

OpenSource toolkits that implement the SAML 2.0 specification for the WebSSO Profile for Service Providers in different programming languages
can help you build your applications and integrations:

- [.NET Framework](https://en.wikipedia.org/wiki/.NET_Framework_version_history) 4.5 or above: [Kentor Authentication Services](https://github.com/KentorIT/authservices#kentor-authentication-services)
- .NET Framework 4 or below: [ComponentSpace SAML 2.0](http://www.componentspace.com/SAMLv20.aspx) - Paid software, licenses start at $299
- Java: [OpenSAML](https://wiki.shibboleth.net/confluence/display/OpenSAML/Home)
- Java/Spring: [Spring Security SAML](/code/java/spring_security_saml)
- PHP: [SimpleSAMLphp](/code/php/simplesamlphp)
- Python: [PySAML2](/code/python/pysaml2)
- Ruby: [Ruby-SAML](https://rubygems.org/gems/ruby-saml)

> Note: Okta doesn't own or maintain these toolkits, though we do provide documentation to help you use them with Okta.

## Reference

### SAML 2.0

- [Assertions and Protocols for the OASIS Security Assertion Markup Language (SAML) V2.0](http://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf)
- [Bindings for the OASIS Security Assertion Markup Language (SAML) V2.0](http://docs.oasis-open.org/security/saml/v2.0/saml-bindings-2.0-os.pdf)
- [Profiles for the OASIS Security Assertion Markup Language (SAML) V2.0](http://docs.oasis-open.org/security/saml/v2.0/saml-profiles-2.0-os.pdf)
