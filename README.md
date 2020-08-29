

# ZPM Demo

##### A Short introduction to the ZPM Package Manager and Registry

This demo uses a docker-compose that starts 3 instances:

* the REGISTRY instance is a private registry that we setup, using the package downloaded from the Intersystems Community.

*  the IRISDEV instance is used to develop and push a new image to our private registry.
  * We first download a package from the community Registry, make modifications to it using VSCode and then package an push this to the private registry.
* the IRISTEST instance connect to our private registry to fetch and install the package



All instances are initially empty. Here we use images of InterSystems Developer community that come with the ZPM package Manager pre-installed.

## Starting the demo

```
docker-compose up
```

The Web ports and URL for the Management Portal of the 3 Instances are as follow:

| Instance | Web Port | (SuperServer) | URL                                        |
| -------- | -------- | ------------- | ------------------------------------------ |
| registry | 6001     | 5001          | http://127.0.0.1:6001/csp/sys/UtilHome.csp |
| iriscode | 6002     | 5002          | http://127.0.0.1:6002/csp/sys/UtilHome.csp |
| iristest | 6003     | 5003          | http://127.0.0.1:6003/csp/sys/UtilHome.csp |

The VSCode which we use in this demo, make use of the Web Port to connect to the Instance, while the IRIS Studio IDE would have to be configured to connect to the Superserver.

Each Instance has volume for durablesys, where its iris system database are located, and a bind mount of a separate subdirectory on our host, so that the /irisdev/app from each instance gets mapped to a different subdirectory in our project:

| Instance | container directory | Host dir   |
| -------- | ------------------- | ---------- |
| registry | /irisdev/app        | ./registry |
| iriscode | /irisdev/app        | ./iriscode |
| iristest | /irisdev/app        | ./iristest |



## Connect with VSCode



## Setup of the private REGISTRY

this section shows

 * how to connect to the default intersystems comunity registry

 * search for packages

 * download and install a package

 * Setting up a private registry

   

Connect to the Registry instance in IRIS with

```
docker exec -it zpmdemo_registry_1 iris session iris
```

Now, enter the ZPM command line tool

```
zpm
```

To View a list of functions, use the "help" command:

```
zpm: USER>help
```

To see all the available packages in the current registry:

```
zpm: USER>search
```

Now, we want to install the "zpm-registry" package on this server:

```
zpm: USER>install zpm-registry
```

The output shows a success:

```
[zpm-registry]  Reload START
[zpm-registry]  Reload SUCCESS
[zpm-registry]  Module object refreshed.
[zpm-registry]  Validate START
[zpm-registry]  Validate SUCCESS
[zpm-registry]  Compile START
[zpm-registry]  Compile SUCCESS
[zpm-registry]  Activate START
[zpm-registry]  Configure START
[zpm-registry]  Configure SUCCESS
[zpm-registry]  Activate SUCCESS
```

Our private Registry is now Live! By default, it has been setup as the username/password authenticated Web Application "**/registry**". It can be reached from other containers at the following URL:

```
http://registry:52773/registry/
```

We could (Should!) define a specific user for authenticating our ZPM clients, but for this demo, _SYSTEM will do.



## Build and Publish a package

In this section, we create a package a publish it onto our private registry. 

Our ./iriscode subdirectory has some code in it, that we will load into the iriscode instance and package:

```
docker exec -it zpmdemo_iriscode_1 iris session iris
```

The docker-compose file has mapped the /irisdev/app directory of the iriscode instance to our local host subdirectory where the installer.cls is located, so we can load it like this:

```
USER> zn "%SYS"
%SYS> do $SYSTEM.OBJ.Load("/irisdev/app/Installer.cls", "ck") 
%SYS> set sc = ##class(App.Installer).setup()
```

In order to create an installable package for ZPM, we need to document it in a module.xml file. The generate method of ZPM allows us to create this file easily. It relies on our source code directory structure to be a tree with following types (this is also the standard export format of Atelier and the isc-dev utility.)

| Subdirectory | Source type                                                  |
| ------------ | ------------------------------------------------------------ |
| ./cls        | ObjectScript classes in .cls form, using subdirectories for class packages |
| ./inc        | includes files as .inc                                       |
| ./mac        | mac routines                                                 |
| ./int        | INT Objectscript                                             |
| ./gbl        | globals in .XML format                                       |
| ./dfi        | DeepSee pivot tables and dashboard definitions in xml format |

Let's run the helper method  "**generate**" to produce the module.xml package definition:

```
zpm: IRISAPP>generate

Enter module folder: /irisdev/app
Enter module name: FirstApp
Enter module version: 1.0.0 =>
Enter module description: First Test at publishing something
Enter module keywords:
Enter module source folder: src =>

Existing Web Applications:
    /csp/irisapp
    /rest-test
    Enter a comma separated list of web applications or * for all: *
    Enter path to csp files for /csp/irisapp:  /irisdev/app/src/csp
Dependencies:
    Enter module:version or empty string to continue:
```

It has produced for us following module.xml:

```
<?xml version="1.0" encoding="UTF-8"?>
<Export generator="Cache" version="25">
  <Document name="firstapp.ZPM">
    <Module>
      <Name>firstapp</Name>
      <Version>1.0.0</Version>
      <Description>First Test at publishing something</Description>
      <Packaging>module</Packaging>
      <CSPApplication CookiePath="/csp/irisapp/" DefaultTimeout="900" DeployPath="${cspdir}irisapp/" MatchRoles=":%DB_IRISAPP:%SQL" PasswordAuthEnabled="1" Recurse="1" ServeFiles="1" ServeFilesTimeout="3600" SourcePath="src/csp" UnauthenticatedEnabled="0" Url="/csp/irisapp" UseSessionCookie="2"/>
      <Resource Name="UnitTests.PKG"/>
      <Resource Name="community.PKG"/>
      <Resource Name="community.objectscript.MacExample.MAC"/>
      <Resource Name="community.objectscript.macroexample.INC"/>
      <CSPApplication CookiePath="/rest-test/" DefaultTimeout="900" DispatchClass="community.objectscript.RESTExample" MatchRoles=":%DB_IRISAPP:%SQL" PasswordAuthEnabled="1" Recurse="1" ServeFiles="2" ServeFilesTimeout="3600" UnauthenticatedEnabled="1" Url="/rest-test" UseSessionCookie="2"/>
      <SourcesRoot>src</SourcesRoot>gene
    </Module>
  </Document>
</Export>
```

If our module had dependencies on other ZPM modules, we could add then now, directly in the module.xml file like this:

```
<Dependencies>
  <ModuleReference>
    <Name>MDX2JSON</Name>
      <Version>2.2.0</Version>
    </ModuleReference>
</Dependencies>
```



Now, we can build this package using the "**load**" command, specifying the directory where our module.xml is located. the "-v" stands for verbose, and is optional:

```
zpm: IRISAPP> load -v /irisdev/app
```

If we have defined some automated unit test, we can execute them now with the "test" command:

```
zpm: IRISAPP>test firstapp
```

Once we have checked that all UnitTests were successfull, it is time to "package" our application before publishing it:

```
package -v firstapp
```



Swith to our Private Resisty

Switch to test registry:

```
repo -r -n registry -url https://test.pm.community.intersystems.com/registry/ -user "test" -pass "test"

```

Switch back to default community repo:

```
repo -r -n registry -url https://pm.community.intersystems.com
or
repo -r -n registry -reset-defaults
```



### Setup Client to work with our private Registry

Before publishing, we need to point the iriscode instance to the correct private registry.

We can Check the current repository:

```
zpm: IRISAPP>repo -list
```

It show the settings for the current InterSystems Community registry, with name "registry"

```
registry
        Source:         https://pm.community.intersystems.com
        Enabled?        Yes
        Available?      Yes
        Use for Snapshots?      Yes
        Use for Prereleases?    Yes
```



Change the remote registry to another one:

```
zpm: IRISAPP> repo -n registry -user _SYSTEM -password Roche -r -url http://registry:52773/registry/
```

Now check the status to verify that your repository is "available" (it means an http request to the endpoint was successful )

```
zpm: IRISAPP>repo -list
```

### Publish the package

```
zpm: IRISAPP>publish
```

we can check if the package is listed with the "search" command

```
zpm: IRISAPP>search

registry http://registry:52773/registry/:
firstapp                            1.0.0
```



### Access the Private Registry from our test server

The final test is really to get this package from a "clean" server and verify it can be installed.

Connect to the iristest server:

```
docker exec -it zpmdemo_iristest_1 iris session iris
```

Change repo

```
USER> zpm
zpm: USER> repo -n registry -user _SYSTEM -password Roche -r -url http://registry:52773/registry/

zpm: USER> repo -list
```

List the Packages

```
zpm: USER>search
```

And Install

```
zpm: USER>install firstapp
```

Finally, verify that the CSP Application serves the hello.csp file. From a browser:

```
http://127.0.0.1:6003/csp/irisapp/hello.csp
```

