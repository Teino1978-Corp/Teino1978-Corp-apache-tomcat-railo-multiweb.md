## Tomcat 
In the Tomcat folder create a `railo/` folder and copy in the contents of the unzipped Railo JARs ZIP file (or from the `WEB-INF/lib/` folder of the unzipped Railo WAR file). In the Tomcat `conf/` folder, edit `catalina.properties` and find the `common.loader` class path. We're going to add the Railo JARs to the common class path so that every web application can have Railo CFML pages. The new `common.loader` definition should look like this (all on one line, no spaces):
`common.loader=${catalina.home}/lib,${catalina.home}/lib/*.jar,${catalina.home}/railo,${catalina.home}/railo/*.jar`

_Note: embedding Railo directly in Tomcat like this means that you will end up with a generated `WEB-INF/` folder in each webroot, containing some Railo files (about 2MB)._ 

Unless you're going to use the default web applications that come with Tomcat, this is a good time to empty the Tomcat `webapps/` folder. You could create a `default/` folder in the Tomcat folder and move everything from `webapps/` to `default/` - this makes it easy to configure the applications again under a new hostname (I'll show this at the end). Next we must configure the Railo Servlet stuff. In the Tomcat `conf/` folder, edit `web.xml`. This is the master web application configuration for the Tomcat server and any web applications you create will inherit from it. At the end of the servlet section, just before the servlet-mapping section, add the following:
``` xml
<servlet>
  <servlet-name>RailoCFMLServlet</servlet-name>
  <description>CFML runtime Engine</description>
  <servlet-class>railo.loader.servlet.CFMLServlet</servlet-class>
  <init-param>
    <param-name>configuration</param-name>
    <param-value>/WEB-INF/railo</param-value>
    <description>Configuration directory</description>
  </init-param>   
  <load-on-startup>1</load-on-startup>
</servlet>   
<servlet>
  <servlet-name>RailoAMFServlet</servlet-name>
  <description>AMF Servlet for flash remoting</description>
  <servlet-class>railo.loader.servlet.AMFServlet</servlet-class>
  <load-on-startup>1</load-on-startup>
</servlet>   
<servlet>
  <servlet-name>RailoFileServlet</servlet-name>
  <description>File Servlet for simple files</description>
  <servlet-class>railo.loader.servlet.FileServlet</servlet-class>
  <load-on-startup>2</load-on-startup>
</servlet>
```
Note that I have prefixed these Servlets with Railo so that you can still deploy Railo WAR-based web apps without conflict (chops to Jamie Krug for this - he renamed his Servlets to have GLOBAL in front of their names). Next, at the end of the servlet-mapping section, just before the filter section, add the following:
``` xml
<servlet-mapping>
  <servlet-name>RailoCFMLServlet</servlet-name>
  <url-pattern>*.cfm</url-pattern>
</servlet-mapping>
<servlet-mapping>
  <servlet-name>RailoCFMLServlet</servlet-name>
  <url-pattern>*.cfml</url-pattern>
</servlet-mapping>
<servlet-mapping>
  <servlet-name>RailoCFMLServlet</servlet-name>
  <url-pattern>*.cfc</url-pattern>
</servlet-mapping> 
<servlet-mapping> 
  <servlet-name>RailoAMFServlet</servlet-name> 
  <url-pattern>/flashservices/gateway/*</url-pattern> 
</servlet-mapping>
```
Note that I have used the default Tomcat Servlet for serving files, per my recent Quick Tip (so you need to set listings to true in the default Servlet definition at the top of the file if you want directory listings). Finally, at end of the file, add `index.cfm` (and `index.cfm`l if you wish) to the list of "welcome files". 
``` xml
<welcome-file-list>
  <welcome-file>index.cfm</welcome-file>
  <welcome-file>index.cfml</welcome-file>
  <welcome-file>index.html</welcome-file>
  <welcome-file>index.htm</welcome-file>
  <welcome-file>index.jsp</welcome-file>
</welcome-file-list>
```
If you emptied the `webapps/` directory above, create an empty folder called `ROOT/` in there so that we'll have at least one web application when we start Tomcat. 

Let's get this party started! We're ready to start Tomcat now. In Terminal, go to the Tomcat bin/ folder and start Tomcat:

`sh startup.sh`

In a few seconds, you'll be able to browse to http://localhost:8080/ and see an empty directory listing. How exciting! Browse to http://localhost:8080/railo-context/admin.cfm and you should see the Railo entry page for the Server and Web Administrators. Login to each and set an initial password. At this point, the Server Administrator data is actually stored in `{tomcat}/railo/railo-server/` and the Web Administrator data is stored in `{tomcat}/webapps/ROOT/WEB-INF/railo/`. Settings in the Server Administrator cascade down into all the Web Administrators associated with the Tomcat server - and the Server Administrator can determine what features can be changed in those Web Administrators. You can stop Tomcat now (by switching to the org.apache.catalina.startup.Bootstrap application and selecting Quit from the menu). The `shutdown.sh` script does not reliably shut Tomcat down for a number of web applications.

Adding websites The first step is always to add a new domain name for each site to your local `etc/hosts` file so all those domains resolve to 127.0.0.1. Let's assume you have `web1.local` and `web2.local` defined there. Let's also assume that the desired webroots for these sites are `~/Documents/sites/web1.local/www/` and `~/Documents/sites/web2.local/www/`. In the Tomcat `conf/` folder, edit server.xml and find the Host definition for localhost (around line 126). You can add your new host definitions below that, as follows:
``` xml
<Host name="web1.local" appBase="webapps" unpackWARs="true" autoDeploy="true" xmlValidation="false" xmlNamespaceAware="false">
  <Context path="" docBase="/Users/yourname/Documents/sites/web1.local/www"/>
</Host>
<Host name="web2.local" appBase="webapps" unpackWARs="true" autoDeploy="true" xmlValidation="false" xmlNamespaceAware="false">
  <Context path="" docBase="/Users/yourname/Documents/sites/web2.local/www"/>
</Host>
```
Now start Tomcat again and you'll see a `WEB-INF/` folder created inside each of those two sites' `www/` folders. Browse to http://web1.local/railo-context/admin/web.cfm, set a new password and login. This is the Web Administrator for the web1.local website. Similarly, http://web2.local/railo-context/admin/web.cfm is the Web Administrator for the web2.local website. The Server Administrator accessible from http://web1.local/railo-context/admin/server.cfm is the same, shared Server Administrator under localhost or web2.local. You now have a shared hosting environment on your local computer with each 'account' having its own full administrator console! Those default Tomcat applications Remember we moved those default applications (docs, examples, host-manager, manager, ROOT)? You can easily make them accessible again as a new website. Add `tomcat.examples` to your hosts file (resolving to 127.0.0.1) and then add the following to Tomcat's `server.xm`l file:
``` xml
<Host name="tomcat.examples" appBase="default" unpackWARs="true" autoDeploy="true" xmlValidation="false" xmlNamespaceAware="false">
</Host>
```
Note the `appBase` is a different top-level folder in Tomcat. When you next restart Tomcat, you'll see Railo's files added to the `WEB-INF/` in each of those five web applications but you'll be able to browse to http://tomcat.examples/docs for example (and view the Tomcat documentation).