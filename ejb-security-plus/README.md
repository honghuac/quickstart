ejb-security-plus:  Using client and server side interceptors to supply additional information for authentication before EJB calls.
====================
Author: Darran Lofthouse  
Level: Advanced  
Technologies: EJB, Security  
Summary: Demonstrates how interceptors can be used to supply additional information to be used for authentication before EJB calls.  
Target Product: WildFly  
Source: <https://github.com/wildfly/quickstart/>  

What is it?
-----------

By default, when you make a remote call to an EJB deployed to the application server, the connection to the server is authenticated and any request received over this connection is executed as the identity that authenticated the connection. The authentication at the connection level is dependent on the capabilities of the underlying SASL mechanisms.

Rather than writing custom SASL mechanisms or combining multiple parameters into one this quickstart demonstrates how username / password authentication can be used to open the connection to the server and subsequently supply an additional security token for authentication before the EJB invocation. This is achieved with the addition of the following three components: 

1. A client side interceptor to pass the additional token to the remote server.
2. A server side interceptor to receive the security token and ensure this is passes to the JAAS domain for verification.
3. A JAAS LoginModule to perform authentication taking into account the authenticated user of the connection and the additional security token.
 
The quickstart then makes use of a single remote EJB, `SecuredEJB` to verify that the propagation and verification of the security token is correct and a `RemoteClient` standalone client. 

### SecuredEJB

For this quickstart the `SecuredEJB` only has the following method: -

    String getPrincipalInformation();

This method is used to confirm the identity of the authenticated user, on the client side changing either the users password or the additional authentication token will cause access to this method to be unavailable.  The output from a successfull call to this method will typically look like: -

	[Principal={quickstartUser}]

### RemoteClient

Finally there is the `RemoteClient` stand-alone client. The client demonstrates how a Principal with a custom credential can be set using the `SecurityContextAssociation` and how this can subsequently be used by a `CallbackHandler` for the username/password authentication required for the SASL authentication and subsequently how the additional token can be picked up by a client side interceptor and passed to the server with the invocation.


Note on EJB client interceptors
-----------------------

JBoss Enterprise Application Platform 6.1 allow client side interceptors for EJB invocations. Such interceptors are expected to implement the `org.jboss.ejb.client.EJBClientInterceptor` interface. User applications can then plug in such interceptors in the 'EJBClientContext' either programatically or through the ServiceLoader mechanism.

- The programmatic way involves calling the `org.jboss.ejb.client.EJBClientContext.registerInterceptor(int order, EJBClientInterceptor interceptor)` API and passing the 'order' and the 'interceptor' instance. The 'order' is used to decide where exactly in the client interceptor chain, this 'interceptor' is going to be placed.
- The ServiceLoader mechanism is an alternate approach which involves creating a `META-INF/services/org.jboss.ejb.client.EJBClientInterceptor` file and placing/packaging it in the classpath of the client application. The rules for such a file are dictated by the [Java ServiceLoader Mechanism](http://docs.oracle.com/javase/6/docs/api/java/util/ServiceLoader.html). This file is expected to contain in each separate line the fully qualified class name of the EJB client interceptor implementation, which is expected to be available in the classpath. EJB client interceptors added via the ServiceLoader mechanism are added to the end of the client interceptor chain, in the order they were found in the classpath.

This quickstart uses the ServiceLoader mechanism for registering the EJB client interceptor and places the `META-INF/services/org.jboss.ejb.client.EJBClientInterceptor` in the classpath, with the following content:

	# EJB client interceptor(s) that will be added to the end of the interceptor chain during an invocation
	# on EJB. If these interceptors are to be added at a specific position, other than last, then use the
	# programmatic API in the application to register it explicitly to the EJBClientContext

	org.jboss.as.quickstarts.ejb_security_plus.ClientSecurityInterceptor


System requirements
-------------------

All you need to build this project is Java 8.0 (Java SDK 1.8) or better, Maven 3.1 or better.

The application this project produces is designed to be run on JBoss WildFly 10.0.1.Final or later.

Configure Maven
---------------

If you have not yet done so, you must [Configure Maven](../README.md#mavenconfiguration) before testing the quickstarts.

Prerequisites
-------------

_Note_: Unlike most of the quickstarts, this one requires JBoss WildFly 10.0.1.Final or later.

This quickstart uses the default standalone configuration plus the modifications described here.

It is recommended that you test this approach in a separate and clean environment before you attempt to port the changes in your own environment.


Configure the JBoss WildFly server
---------------------------

These steps assume that you are running the server in standalone mode and using the default standalone.xml supplied with the distribution.

_NOTE - Before you begin:_

1. If it is running, stop the JBoss WildFly Server.
2. Backup the file: `WILDFLY_HOME/standalone/configuration/standalone.xml`
3. After you have completed testing this quickstart, you can replace this file to restore the server to its original configuration.

1. Start the WildFly Server by typing the following: 

        For Linux:  WILDFLY_HOME/bin/standalone.sh 
        For Windows:  WILDFLY_HOME\bin\standalone.bat
2. Open a new command line, navigate to the root directory of this quickstart, and run the following command, replacing JBOSS_HOME with the path to your server:

        WILDFLY_HOME/bin/jboss-cli.sh --connect --file=configure-security-domain.cli
This script adds the `quickstart-domain` domain to the `security` subsystem in the server configuration and configures authentication access. You should see the following result when you run the script:

        #1 /subsystem=security/security-domain=quickstart-domain:add(cache-type=default)
        #2 /subsystem=security/security-domain=quickstart-domain/authentication=classic:add
        #3 /subsystem=security/security-domain=quickstart-domain/authentication=classic/login-module=DelegationLoginModule:add(code=org.jboss.as.quickstarts.ejb_security_plus.SaslPlusLoginModule,flag=optional,module-options={password-stacking=useFirstPass})
        #4 /subsystem=security/security-domain=quickstart-domain/authentication=classic/login-module=RealmDirect:add(code=RealmDirect,flag=required,module-options={password-stacking=useFirstPass})
        The batch executed successfully.
        {"outcome" => "success"}

Add the Application Users
---------------

This quickstart is built around the default `ApplicationRealm` as configured in the JBoss WildFly server distribution. Using the add-user utility script, you must add the following user to the `ApplicationRealm`:

| **UserName** | **Realm** | **Password** | **Roles** |
|:-----------|:-----------|:-----------|:-----------|
| quickstartUser| ApplicationRealm | quickstartPwd1!| User |

This user is used to both connect to the server and is used for the actual EJB invocation.

For an example of how to use the add-user utility, see instructions in the root README file located here: [Add User](../README.md#addapplicationuser).

Start JBoss WildFly Server
-------------------------

1. Open a command line and navigate to the root of the JBoss server directory.
2. The following shows the command line to start the server with the web profile:

		For Linux:   WILDFLY_HOME/bin/standalone.sh
		For Windows: WILDFLY_HOME\bin\standalone.bat


Build and Deploy the Quickstart
-------------------------

_NOTE: The following build command assumes you have configured your Maven user settings. If you have not, you must include Maven setting arguments on the command line. See [Build and Deploy the Quickstarts](../README.md#buildanddeploy) for complete instructions and additional options._

1. Make sure you have started the JBoss Server as described above.
2. Open a command line and navigate to the root directory of this quickstart.
3. Type this command to build and deploy the archive:

		mvn clean package wildfly:deploy

4. This will deploy `target/wildfly-ejb-security-plus.jar` to the running instance of the server.


Run the client
---------------------

The step here assumes you have already successfully deployed the EJB to the server in the previous step and that your command prompt is still in the same folder.

1.  Type this command to execute the client:

		mvn exec:exec


Investigate the Console Output
----------------------------

When you run the `mvn exec:exec` command, you see the following output.

    * * * * * * * * * * * * * * * * * * * * * * * * * * * * * *
    [Principal={quickstartUser}]
    * * * * * * * * * * * * * * * * * * * * * * * * * * * * * *

This output is only displayed when the client has supplied the correct username, password and authentication token combination.

Re-Run the client
---------------------

You can edit the class `RemoteClient` and update any one of username, password or authentication token.

1.  Type this command to execute the client:

		mvn compile exec:exec

At this point instead of the message shown above you should see a failure.

Undeploy the Archive
--------------------

1. Make sure you have started the JBoss Server as described above.
2. Open a command line and navigate to the root directory of this quickstart.
3. When you are finished testing, type this command to undeploy the archive:

		mvn wildfly:undeploy


Remove the Security Domain Configuration
----------------------------

You can remove the security domain configuration by running the  `remove-security-domain.cli` script provided in the root directory of this quickstart.

1. Start the JBoss WildFly Server by typing the following: 

        For Linux:  WILDFLY_HOME/bin/standalone.sh
        For Windows:  WILDFLY_HOME\bin\standalone.bat
2. Open a new command line, navigate to the root directory of this quickstart, and run the following command, replacing JBOSS_HOME with the path to your server:

        WILDFLY_HOME/bin/jboss-cli.sh --connect --file=remove-security-domain.cli 
This script removes the `quickstart-domain` security domain from the `security` subsystem in the server configuration. 

Run the Quickstart in JBoss Developer Studio or Eclipse
-------------------------------------

You can also start the server and deploy the quickstarts from Eclipse using JBoss tools. For more information, see [Use JBoss Developer Studio or Eclipse to Run the Quickstarts](../README.md#useeclipse) 


Debug the Application
------------------------------------

If you want to debug the source code or look at the Javadocs of any library in the project, run either of the following commands to pull them into your local repository. The IDE should then detect them.

	mvn dependency:sources
	mvn dependency:resolve -Dclassifier=javadoc
