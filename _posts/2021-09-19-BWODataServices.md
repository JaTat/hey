---
layout: post
title: OData Services in SAP BW/4 from a HANA table
image:
  path: /assets/img/blog/OData/social-media-5187243_1920.png
  srcset:
    1920w: /assets/img/blog/OData/social-media-5187243_1920.png
    960w:  /assets/img/blog/OData/social-media-5187243_1280.png
    480w:  /assets/img/blog/OData/social-media-5187243_640.png.jpg
description: >
  Creating an OData service from a SAP HANA table in my SAP BW/4
grouped: true
---

Hey there! Imagine you want to offer your carefully modelled data in your SAP BW to non-SAP tools. The BW4/HANA provides you three ways how to do this [(SAP Reference)](https://help.sap.com/viewer/107a6e8a38b74ede94c833ca3b7b6f51/2.0.7/en-US/4a12e84976df1b42e10000000a42189c.html):
1. OpenHub - Replicate data from the BW by writing it to databases or files
2. Via the HANA database itself by generating HANA views (For how to access the HANA from Python see my last blog, [link](https://jantatzel.com/2021-08-30-PythonToBW/))
3. Via OData APIs

Today I am exploring the OData Services in my SAP BW4/HANA 2.0. There are two ways to expose an OData service. 

1. An OData service directly from the HANA Database using the SAP HANA XS application server. This is not restricted to the BW application but can be done from any HANA database with the XS application server.
2. From the BW:
	- Using the Operational Data Provisioning Framework (ODP)
	- Using Easy Queries 
Using OData as a communication protocol for ODP is meant for similar use cases as the OpenHub functionality. It offers the full range of functionality of ODP such as direct data access, operational delta queue, and delta extraction with initial full load.

I will build my service directly from the HANA database using the SAP HANA platform services.

# What is OData

[OData](https://www.OData.org/) is Microsoft's attempt on a Protocol that basically allows SQL operations such as filtered selects and joins via http(s):
- It is built on REST standards
- It is currently in its 4th iteration.
- It is maintained by a non-profit organization called [OASIS](https://en.wikipedia.org/wiki/OASIS_(organization)). 

With an OData API the client can specify exactly which part of a resource it wants. This makes a lot of sense if the resource is a structured entity such as a table or a data object with multiple fields. Then the client can use methods similar to SQL statements such as selecting certain columns, filtering, ordering, even joining of two entities is possible. This means the client does not overfetch data, i.e. receives more than it wanted, and probably can chew. It also prevents underfetching, resulting in many API calls to the backend by the client.
## REST

Since OData claims to follow the REST guidelines, which is an abbreviation of Representational State Transfer,  it is fair to expect some things. REST is the de facto architecture style for developing APIs for Web applications. This is because the REST guidelines were developed to solve a lot of challenges of the internet, i.e. for a complex client server systems with potentially multiple layers of proxies and reverse proxies in between and any number of protocols. REST was developed in the context of hypermedia systems, these are rich information systems meant for non-linear consumption. There is no official REST standard but in general APIs are said to be RESTful if they satisfy the following non functional requirements:
- Client–server architecture: Separating the UI from data storage, retrieval and preparation for messaging/ UI consumption e.g. serializing into a set representation that can be interpreted by the widest number of clients
- Statelessness: The server responding to a request needs no information about the session. Each request can be fulfilled in insolation and after the request has been fulfilled, i.e. the packet has been sent the server can forget the request.
- Cacheability: Since REST assumes many layers between the client and the server, caching, i.e. storing information temporarily in another layer than the responding server, can severely increase response times. To prevent data staleness responses have to be explicite about their cacheability.
- Code on demand: Servers can extend or customize the functionality of a client by transferring executable code, e.g. with JavaScript
- Uniform interface: Decoupling is achieved by separating the resource from the representation, i.e. how data is represented can differ from how data is stored on the server; the representation sent to the client, including all metadata has to be enough for the client to transform the received representation, self descriptive messages including information to describe how to process the message and finally **discoverability** of resources, a client should be able to discover all available resources from an initial URI


## Adoption

In the Microsoft and SAP system OData is available in many products such as Microsoft 365 (Office, Teams, ...) via Microsoft Graph [Graph Explorer](https://developer.microsoft.com/en-us/graph/graph-explorer), several other products such as Azure DevOps and PowerBI.

OData is used by SAP to serve data to its Fiori Web Apps. This means you can find it in the entire Business Suite.

Besides these two heavy weights adoption seems limited. Netflix provided access to its video catalogs with an OData Api but discontinued it in 2013 , so did eBay ([Source](https://www.ben-morris.com/netflix-has-abandoned-OData-does-the-standard-have-a-future-without-an-ecosystem/)).

## Alternatives

As always in the world of tech there are alternatives available. Facebook promotes an API standard called [GraphQL](https://graphql.org/) for similar tasks (for a discussion see [here](https://jeffhandley.com/2018-09-13/graphql-is-not-OData) ) and Oracle has something called [ORDS - Oracle Rest data services](https://www.oracle.com/database/technologies/appdev/rest.html)


#  Activating an OData Service for a single HANA Table

As always I prefer exploring over huge upfront research. And as with my other blog posts I came to a point where I had to do a fair bit of research. My main challenges were authorization and no clear understanding of the various components of the BW/4 and HANA platform. Again the following are notes of my explorations annotated with later learnings.

## Creating a OData service in HANA modelling tools

I start in the HANA modelling tools since this is also the IDE for the BW system. The later me will learn that there is another way using something called the Web IDE and a completely different set of Web development services but oh well for now I use what I know.

I create a new repository package, OData, in the eclipse HANA development perspective under my Tenant DB:

![Repo](/assets/img/blog/OData/CreateRepository.png)

Following [this](https://blogs.sap.com/2018/07/02/how-to-expose-a-hana-table-via-OData/) Blogpost I now have to create a number of files:

They are all created similiarly by right clicking the package, New, Other and choosing the correct wizard.

![Repo](/assets/img/blog/OData/CreateArtifacts.png)


## .xssqlcc - Anonymous SQL Access Configuration
This is the configuration file for the sql connection.

Current me: Following the blog I changed the filename to *ANONUSER*.

I double clicked on the file and added *{ “description” : “Anonymous SQL connection” }*. The I activated the file with Ctrl+F3.

I got an error during activation and had to apply a quick fix by right clicking on the error message in the eclipse console and change the file encoding.

Later me: Quote from the [SAP help](https://help.sap.com/viewer/b3d0daf2a98e49ada00bf31b7ca7a42e/2.0.03/en-US/0d2aa67a44a94b14ae80dc883a4c6419.html):
*In SAP HANA Extended Application Services (SAP HANA XS), you use the SQL-connection configuration file to configure a connection to the database; the connection enables the execution of SQL statements from inside a server-side JavaScript application with credentials that are different to the credentials of the requesting user.* 

The structure of the file is simple. It contains a the description and potentially a mapping to a role configuration. The file can only be accessed from within the package and is referenced with *package::filename*

## .xsaccess - Access authorization

Next up I use the same steps as above to create a .xsaccess file. This file defines the authorization settings and also has additional parameters such as cache_control etc.

Quoting from the SAP Help [Link](https://help.sap.com/viewer/400066065a1b46cf91df0ab436404ddc/2.0.02/en-US/5fe3b123826d4503aa66eb955a002821.html): *The application-access file enables you to specify who or what is authorized to access the content exposed by a SAP HANA XS application package and what content they are allowed to see. For example, you use the application-access file to specify if authentication is to be used to check access to package content and if rewrite rules are in place that hide or expose target and source URLs.*

The file has no name and is formatted in json. Parameters can also be obtained from the SAP help

The *anonymous_connection"* line ties back to the file we created above, where OData is the package and anonuser the file.

~~~json
{
	"exposed": true, 
	"authentication": null,

	"mime_mapping": [{
		"extension": "jpg",
		"mimetype": "image/jpeg"
	}],
	"prevent_xsrf" : false,
	"force_ssl": false,
	"enable_etags": true,
	"anonymous_connection": "OData::anonuser",
	"cors": [{
		"enabled": true,
		"allowMethods": ["GET"],
		"allowOrigin": ["*"]
	}],
	"allowHeaders": [
		"Accept",
		"Authorization",
		"Content-Type",
		"X-CSRF-Token",
		"Access-Control-Allow-Origin"
	],
	"exposeHeaders": [
		"x-csrf-token"
	],
	"cache_control": "no-cache, no-store"
}
~~~

Parameters:

*cors*: I only allow **read** access from domains other than the domain of my XS HANA application server. I could further narrow down the allowed domains by "allowOrigin" e.g. to the domain of a WebApp that is supposed to query the data from my HANA.

Note: *prevent_xsrf*, *force_ssl* and *authentication* are  important security features and should be used in production.

- *prevent_xsrf*  checks for a valid security token of the client vs. the generated token of the HANA XS Server. It helps to prevent cross-site request-forgery (XSRF) attacks. In these attacks a user is tricked on clicking on a hyperlink to a Website that performs actions on the users behalf.

- *force_ssl* rejects requests via HTTP and thereby ensure the encryption of all data sent.

- *authentication* must be set to *method: Form* according to the documentation to redirect any user requests to a login page.

These options require additional configuration as detailed [in the SAP Help](https://help.sap.com/viewer/400066065a1b46cf91df0ab436404ddc/2.0.02/en-US/a9fc5c220d744180850996e2f5d34d6c.html).


### .xsapp - Application Description
Next up is the .xsapp file. I create the file .xsapp and add curly brackets:
~~~json
{}
~~~
Later me:
The purpose of the file is *The application descriptor is the core file that you use to indicate an application's availability within SAP HANA XS. The application descriptor marks the point in the package hierarchy at which an application's content is available to clients. The application-descriptor file has no contents and no name; it only has the file extension .xsapp. The package that contains the application-descriptor file becomes the root path of the resources exposed by the application you develop.* [Source](https://help.sap.com/viewer/400066065a1b46cf91df0ab436404ddc/2.0.02/en-US/0549991ea9984835b8c06c12123b7fe2.html) Since it defines the root it also defines the root URL where your application will be reachable. In my case this will therefore be *\<host>:\<port>/OData/*



### .xsOData - OData Service Specification

Now I need the actual OData description. I do this by adding the .xsOData file and specifying which table(s) I want to expose as service.

~~~json
service { "SAPHANADB"."RSDIOBJT" as "IOBJT"; }
~~~

- *IOBJT* is the alias for the table name in the OData service.

Tip: I use the auto code completion in the SQL editor (ctrl + space) to get the "SCHEMA"."TABLE" specification.


## Run OData Service

A right click in the text editor of the .xsOData file, opens the context menu. Here I click on Run As 1 XS Service.

![Run As](/assets/img/blog/OData/Run.png)


 The default browser opens with the URL 
https://\<Hostname>:43\<InstanceNumber>//OData/\<Alias>.xsOData/

In my SAP CAL instance this results in the following URL:

https://vhcalhdbdb:4302//OData/BWIobjt.xsOData/

But bummer I get a 403 error which tells me that I have an authorization issue.


## A lot of trial and error to solve the authorization issue

Adding a Database user to an OData service can generally be done with the XS Admin Tool. In my CAL system I can access it via https://vhcalhdbdb:4302/sap/HANA/xs/admin/#/. But which is the right user to login to the admin tool?

### Accessing the HANA XS Admin Tool

In the logon screen I try all my configured users but none works. A quick google search for how to maintain the XS users tells me that there is a Command Line tool on the Application server named *xs*, see the [SAP Help](https://help.sap.com/viewer/4505d0bdaf4948449b7f7379d24d0f0d/2.0.03/en-US/addd59069e6f444ca6ccc064d131feec.html) for the reference. I logon via putty to my application server but cannot locate the xs executable. I locate a configuration file named xsconfig via a find -name "*xs*" and it reveals that *XSA_ADMIN* is the user.

Later me:
Little did I know that there is a completely separate web development stack called XS**A** available and activated in my SAP CAL BW/4.

![Application Server xsconfig](/assets/img/blog/OData/xsconfig.png)

Trying to login with the XSA_ADMIN user results in the same problem: 403. Okay so back to square 1 and some more googling.

### Access with the system user

I try again all other users and note that with the system user I get a not authorized instead of wrong credentials. So I guess I have to grant additional rights to the SYSTEM user.

In my Tenant Database configuration in the HANA modelling tools, I grant all xs related roles to the System user by navigating to Security --> Users --> System and clicking on add roles.

![Grant all xs roles](/assets/img/blog/OData/Grant.png)

Most likely the WebDispatcherAdmin Role Should be enough since *The role sap.HANA.xs.wdisp.admin::WebDispatcherAdmin enables full access to the SAP HANA Web Dispatcher Administration tool, which the administrator uses to maintain secure inbound communication, for example, to enable SSL/TLS connections between an ABAP system and an SAP HANA XS application.* according to the [Reference](https://help.sap.com/viewer/6b94445c94ae495c83a19646e7c3fd56/2.0.02/en-US/9fdfbe396db9496c993779db91df01aa.html)

I try to login again but receive the same 403 error. Weird....

After a coffee I have an idea, what if the Admin tool tries to login to the System database?

I grant the System user all xs related rights on the system database as above and finally I can login. But now I only see the services exposed on the system database and not the OData service created before. So again back to square 1 and of course more research.

### Enable the tenant database.

Later me: I believe what follows is a complete detour. I have been using the now deprecated HANA XS service so far. What follows now is how I discover the HANA XS**A** service. The actual solution is 

I find out that one has to enable a Tenant database either with the Command Line Tools as described [here](https://help.sap.com/viewer/6b94445c94ae495c83a19646e7c3fd56/2.0.04/en-US/946072aa37274912aa7f858151dd5e65.html) or the XS HANA ADVANCED Cockpit a Web Application that in my case is located here:
https://vhcalhdbdb.dummy.nodomain:51024/cockpit#/xsa/overview

Here I note that my Tenant is not enabled (the green enabled label in enabled for XSA was missing) and I enable it under Actions --> Enable by entering the user credentials of my Tenant and System database user.

Since I am already here, I create a new Org under Organizations on the left and map the Tenant Database to the Org (JanOrg) again under Actions.

![Enable Tenant](/assets/img/blog/OData/MapTenant.png)
Unfortunately in the XS Admin Tool I still see the System database services.

### Editing the xsengine.ini

This blog post [here](https://answers.sap.com/questions/13133354/how-to-access-sap-HANA-20-tenant-database-xs-admin.html) gave me the idea that I have to alter my xsengine.ini configuration file in order for the admin tools to be able to access the tenant database. This is described in more detail in the [SAP Help](https://help.sap.com/viewer/78209c1d3a9b41cd8624338e42a12bf6/1.0.12/en-US/af24ce38929b4f1fa934a28147fc7641.html).

I have to open the administration editor of in the System tab of the HANA Development Perspective and choose the configuration tab. I do this in the context of my tenant database.

There I find the xsengine.ini with the following settings under publicurl:

![xsengine configuration](/assets/img/blog/OData/xsengineini.png)

Instead of adding new public URLs I double click the entry and add database A4H which was empty before. Now a green icon in the column database indicates that the database is connected:

![Green light](/assets/img/blog/OData/green.png)

As described in the SAP help I run:

~~~SQL

ALTER SYSTEM ALTER CONFIGURATION ('xsengine.ini', 'database', '<tenant_DB_name>') SET ('public_urls', 'http_url') = 'http://<virtual_hostname>:80<instance>' WITH RECONFIGURE;

ALTER SYSTEM ALTER CONFIGURATION ('xsengine.ini', 'database', '<tenant_DB_name>) SET ('public_urls', 'https_url') = 'https://<virtual_hostname>:43<instance>' WITH RECONFIGURE;

~~~

with 

- Tenant Database: A4H
- Virtual_hostname: vhcalhdbdb
- instance: 02

I run this Query in the system database and the tenant database since its not specified in the reference. In the tenant database I get an error telling me that I do not have to specify the tenant database name, so I exclude \<tenant_DB_name> from the query.

## Configure the SQL user in the XS ADMIn Tool

Finally I see my OData Service in the XS Admin tool. I can configure the Database user with a click on edit. I add the SAPHANADB user to the anonymous sql connection.

![Finally the db user for the OData Service can be configured](/assets/img/blog/OData/ODataXS.png)


## Accessing the OData service

In the browser I can now access the OData service under:

https://vhcalhdbdb:4302//OData/BWIobjt.xsOData/

and the metadata of the service under

https://vhcalhdbdb:4302//OData/BWIobjt.xsOData/$metadata

So finally I can query all InfoObject descriptions in my BW/4 with a get request to https://vhcalhdbdb:4302//OData/BWIobjt.xsOData/IOBJT?$filter=LANGU%20eq%20%27E%27

This returns English descriptions as xml:

![Get Data](/assets/img/blog/OData/ODataQuery.png)

## Conclusion:

It was again a bit harder than I thought. Getting the OData service to work involved a lot of trial and error and googling. Although with a properly set up system creating an OData service can be very fast since it only involves the steps described under Activating an OData Service for a single HANA Table.


## Note from future me:

SAP HANA XS, extended application services, are deprecated as of SAP HANA 2.0 SPS02. Currently we are at SPS 05. The services were superseded by HANA XSA the 
**advanced** application services. This is a set of tools to allow Web applications natively on HANA. XSA uses cloud foundry to allow microservice style, containerized web services and comes with a full stack of development tools, such as a Web IDE. The administration cockpit I discovered above is part of the stack. Apparently the integration of XSA with SAP BW/4 is still a bit patchy. For example HANA containers spawned by XSA are not aware of schemas defined in BW/4, see [this](https://blogs.sap.com/2020/02/21/bw-4-hana-mixed-scenarios-with-native-hana-hdi-containers/) blog post. See [here](https://blogs.sap.com/2017/09/04/xs-advanced-for-not-so-dummies/) for an introduction to XSA. XSA sounds cool so I plan to have a look at it in the near future.
--------

###### The screenshots are my own, the header picture is a license free picture from pixabay
