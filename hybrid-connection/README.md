# Azure Hybrid Connections

We use an Azure Hybrid Connection to connect to an on-prem SQL Server from within the Azure AppService PaaS.

## Enabling Hybrid Connection

Hybrid Connections require at least a "Standard" App Service plan.
The `az` client does not yet support setting up hybrid connections.  ARM or Terraform must be used to fully automate this solution.

For now we just enable the Hybrid Connection from the Azure portal.  `App Service -> Networking -> Hybrid Connections`

```
Hybrid Connection Name: prototype-HybridConnection (This is the name of the new HC resource)
Endpoint Host: Obtain from AK
Endpoint Port: Obtain from AK

Servicebus namespace: Create new
Location: Us West 2
Name: prototype-ServiceBus 
```

**\*Note**: The "Endpoint Host" is confusing. It is the hostname as seen from the machine where the HCM is installed. We'll install the HCM on our SQL Server VM.  "localhost" is reserved, so get the correct hostname on the VM with Powershell: `hostname`.


### Install Hybrid Connection Manager

This client is installed on the machine that the AppService app is accessing. In our case, some external SQL Server.
On the Hybrid Connections screen there is a "Download connection manager" link. Use it.

Launch the "Hybrid Connection Manager UI" program and "Configure new Hybrid Connection". The program will ask for your Azure login and walk you through setup.

### Deployment for testing

I'm deploying to a short-lived AzureApps resource. I created it and deploy to it as described in `/appservice` folder.
Under "Application Settings" I added a new "App Setting" `deployment_branch` and set it to my feature branch `hcm-prototype-updates` so I could deploy from a branch other than master.

### Hybrid/SQL Connection Strings

SQL Auth is what we use for now. I manually added a new SQL Server user using SSMS.
In the future we will investigate other authentication options.

You can test your connection string locally on the "remote" SQL Server in powershell.

```ps1
$connstr = "Server=SQLServer,1433;Database=AKTestDataBase;User ID=<changeme>;Password=<changeme>"
$sqlconn = New-Object System.Data.SqlClient.SqlConnection($connstr) 
$sqlconn.Open();
$sqlconn.Close();
```
**\*Note**: the `User ID` and `Password` just above are SQL Server Auth credentials that you'll have to set up.

**Further, the connection string is _identical_ for local/VM and Hybrid/PaaS connections.** This is the magic of Hybrid Connections.

### Hybrid/WebService Connection

Hybrid connection can give us access to on-prem web services from Azure.  The web service must be accessible from the server running the Hybrid Connection Manager.  To access a service, use the internal domain name for the Endpoint Host, set port 443 for https. 


## SQL Authentication over Hybrid Connection

Unfortunately, **only SQL Server Auth is supported over the Hybrid Connection**.  This means we cannot use Active Directory style authentication to an on-prem SQL Server over the Hybrid Connection. This has been verified with Azure and SQL Server support.

## Things you cannot do with Hybrid Connections

- mounting a drive
- using UDP
- TCP over dynamic ports
- LDAP (b/c it sometimes requires UDP)
- Active Directory Support

## Hybrid Connection underlying technology

Please see the following pages for further documentation from Microsoft.

 [Azure Relay Documentation](https://docs.microsoft.com/en-us/azure/service-bus-relay/relay-what-is-it)
> In the relayed data transfer pattern, an on-premises service connects to the relay service through an outbound port and creates a bi-directional socket for communication tied to a particular rendezvous address. The client can then communicate with the on-premises service by sending traffic to the relay service targeting the rendezvous address. The relay service then "relays" data to the on-premises service through a bi-directional socket dedicated to each client. The client does not need a direct connection to the on-premises service, it is not required to know where the service resides, and the on-premises service does not need any inbound ports open on the firewall.

> The key capability elements provided by Relay are bi-directional, unbuffered communication across network boundaries with TCP-like throttling, endpoint discovery, connectivity status, and overlaid endpoint security. The relay capabilities differ from network-level integration technologies such as VPN, in that relay can be scoped to a single application endpoint on a single machine, while VPN technology is far more intrusive as it relies on altering the network environment.

[AppService Hybrid Connection Documentation](https://docs.microsoft.com/en-us/azure/app-service/app-service-hybrid-connections)
> The connection uses TLS 1.2 for security and SAS keys for authentication/authorization.

[Hybrid Protocol Documentation](https://docs.microsoft.com/en-us/azure/service-bus-relay/relay-hybrid-connections-protocol)


## Whitelists

If our on-prem service lives in an environment where we must whitelist outbound connections, then we must whitelist various Azure/Hybrid things.

Ref:
- https://docs.microsoft.com/en-us/azure/service-bus-relay/relay-hybrid-connections-protocol
- https://blogs.msdn.microsoft.com/waws/2017/06/30/things-you-should-know-web-apps-and-hybrid-connections/#WhiteList

Machine w/ HCM must whitelist **all of**:

1. "Service Bus endpoint URL" : find this value in the HCM details screen labeled as "Service Bus Endpoint" (`prototype-servicebus.servicebus.windows.net` for our demo)

2. "Service Bus Gateways" : these are 128 different servers.  See discussion at "things you should know" link above.  128 different whitelists will have to be added for `G0-prod-[stamp]-010-sb.servicebus.windows.net` -> `G127-prod-[stamp]-010-sb.servicebus.windows.net`. (Hopefully we can use wildcards!?). `[stamp]` can be found with `nslookup`, again see doc above.

3. "Azure IP Address" : find this value in the HCM details screen labeled "Azure IP Address"

Note: various Hybrid docs mention that HCM must have "outbound access to Azure over ports 80 and 443", but we have verified that only 443 is used and need be opened.


A good way to test outbound access in Windows: `Test-NetConnection -ComputerName prototype-servicebus.servicebus.windows.net -Port 443`

## HCM Logging

To debug the HCM connection use the Windows Event Viewer.  Logs are under `Applicaiton and Service Logs / Microsoft / ServiceBus / Client`.

## Security Review

18F performed a security review of the Azure Hybrid Connection, the results are here in [Azure_Hybrid_Connection_Security_Review.pdf](./Azure_Hybrid_Connection_Security_Review.pdf)


## TODO: overcome lost hybrid connections issue

We have noticed the Hybrid Connection Manager lose connections. The problem seemed to originate with a failed DNS lookups. Why the HCM service didn’t automatically recover after the DNS was restored is a mystery.  There isn't an "automatically reconnect" option in the HCM.

The log error is `Name resolution for the name protowebapi.servicebus.windows.net timed out after none of the configured DNS servers responded`

Find HCM logs through Windows Event Logs.  Under Windows events, there is a separate application-level log for the service bus.  The error above is acutally in the "Windows Logs -> System" namespace.

**The Manual Fix** for now is to just restart the HCM service.  Use "Services" app and restart "Azure Hybrid Connection Manager Service".

Some people have worked around this with a keepalive script: https://stackoverflow.com/questions/40516417/successfully-established-hybrid-connection-loses-connection-after-20-minutes-re