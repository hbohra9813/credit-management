http://blogs.msdn.com/b/kwill/archive/2011/05/05/windows-azure-role-architecture.aspx. 

*Update Aug 20, 2013: The guest agent process used to be WaAppAgent, and any updates to this guest agent would come at the same time as a Guest OS update.  This has been changed to use 2 guest agents - WaAppAgent and WindowsAzureGuestAgent.  WindowsAzureGuestAgent takes on all of the work that WaAppAgent used to do, and WaAppAgent is now responsible for installing, configuring, and updating WindowsAzureGuestAgent.  This decouples the guest agent from the Guest OS itself and allows updates to the guest agent to be out of band with Guest OS updates.  This is reflected in the chart below by the C2 process. 

  

One of the more common questions I get is about the architecture of an Azure role and which processes are responsible for the various steps in getting an Azure role instance up and running, how they interact, etc.  This blog post will attempt to give a good overview of the major steps that happen when deploying a service, and the core processes that run on an Azure VM. 
   

FProcess Information 

A.      RDFE / FFE is the communication path from the user to the fabric.  RDFE (RedDog Front End) is the publicly exposed API which is the front end to the Management Portal and the Service Management API (ie. Visual Studio, Azure MMC, etc).  All requests from the user go through RDFE.  FFE (Fabric Front End) is the layer which translates requests from RDFE into the fabric commands.  All requests from RDFE go through the FFE to reach the fabric controllers. 

B.      Fabric controller is responsible for maintaining and monitoring all of the resources in the data center.  It communicates with fabric host agents on the fabric OS sending information such as the Guest OS version, service package, service configuration, service state, etc. 

C.      Host Agent lives on the Host OS and is responsible for setting up Guest OS and communicating with Guest Agent (WindowsAzureGuestAgent) in order to drive the role towards goal state and to do heartbeat checks with the guest agent.  If host agent does not receive heartbeat response for 10 minutes, host agent will restart the guest OS. 

C2.     WaAppAgent is responsible for installing, configuring, and updating WindowsAzureGuestAgent.exe. 

D.      WindowsAzureGuestAgent is responsible for: 

a.       Configuring the guest OS including firewall, ACLs, LocalStorage resources, service package and configuration, certificates. 

b.      Setting up the SID for the user account which the role will run under. 

c.       Communicating role status to the fabric. 

d.      Starting WaHostBootstrapper and monitoring it to ensure role is in goal state. 

E.       WaHostBootstrapper is responsible for: 

a.       Reading the role configuration and starting up all of the appropriate tasks and processes to configure and run the role. 

b.      Monitoring all of its child processes. 

c.       Raising the StatusCheck event on the role host process. 

F.       IISConfigurator runs when the role is configured as a Full IIS web role (it will not run for SDK 1.2 HWC roles).  It is responsible for: 

a.       Starting the standard IIS services 

b.      Configuring rewrite module in web config 

c.       Setting up the AppPool for the <Sites> configured in the service model. 

d.      Setting up IIS logging to point to the DiagnosticStore LocalStorage folder 

e.      Configuring permissions and ACLs 

f.       The website resides in %roleroot%:\sitesroot\0 and the apppool is pointed to this location to run IIS. 

G.     Startup tasks are defined by the role model and started by WaHostBootstrapper.  Startup tasks can be configured to run in the Background asynchronously and the host bootstrapper will start the startup task and then continue on to other startup tasks.  Startup tasks can also be configured to run in Simple (default) mode where the host bootstrapper will wait for the startup task to finish running and return with a success (0) exit code before continuing on to the next startup task. 

H.      These tasks are part of the SDK and defined as plugins in the role’s service definition (.csdef).  When expanded into startup tasks the DiagnosticsAgent and RemoteAccessAgent are unique in that they define 2 startup tasks each, one regular and one with a /blockStartup parameter.  The normal startup task is defined as a Background startup task so that it can run in the background while the role itself is running.  The /blockStartup startup task is defined as a Simple startup task so that WaHostBootstrapper will wait for it to exit before continuing.  The /blockStartup task simply waits for the regular task to finish initializing and then it will exit and allow the host bootstrapper to continue.  The reason this is done is so that diagnostics and RDP access can be configured prior to the role processes starting up (this is done via the /blockStartup task), and that diagnostics and RDP access can continue running after the host bootstrapper has finished with startup tasks (this is done via the normal task). 

I.        WaWorkerHost is the standard host process for normal worker roles.  This host process will host all of the role’s DLLs and entry point code such as OnStart and Run. 

J.        WaWebHost is the standard host process for web roles when they are configured to use the SDK 1.2 compatible Hostable Web Core (HWC).  Roles can enable the HWC mode by removing the <Sites> element from the service definition (.csdef).  In this mode all of the service’s code and DLLs run from the WaWebHost process.  IIS (w3wp) is not used and there are no AppPools configured in IIS Manager because IIS is hosted inside of WaWebHost.exe. 

K.      WaIISHost is the host process for role entry point code for web roles using Full IIS.  This process will load the first DLL found which implements the RoleEntryPoint class (this DLL is defined in E:\__entrypoint.txt) and execute the code from this class (OnStart, Run, OnStop).  Any RoleEnvironment events (ie. StatusCheck, Changed, etc) created in the RoleEntryPoint class will be raised in this process. 

L.       W3WP is the standard IIS worker process which will be used when the role is configured to use Full IIS.  This will run the AppPool configured from IISConfigurator.  Any RoleEnvironment events (ie. StatusCheck, Changed, etc) created here will be raised in this process.  Note that RoleEnvironment events will fire in both locations (WaIISHost and w3wp.exe) if you subscribe to events from both processes. 

  


 
