# DRAFT

# Disclaimer

The purpose of this tutorial is to show an example of how to build and install the DotStatSuite .NET Core (also referred to as DotStatSuite Core) applications on a set of Windows servers. It is not intended as a guide on how to set it up securely, and taking any security advice unchecked from this would be exceptionally unwise. Use official documentation or advice from your organization's IT security team.

# Introduction

The purpose of this tutorial is to describe how one could go about building and deploying a simple working set of DotStatSuite Core services in a Windows environment. This means building on a Windows machine, and deploying **to** Windows machines. With this in mind, the scripting language I'll be using is Powershell, and the web server I'll be deploying on is Microsoft's Internet Information Services. However, a lot of the steps and information contained in this tutorial would be the same on other platforms, even if the exact commands might differ. Where possible I'll be noting what might be different.

# The Cast of Characters

We will be building (from source) and deploying three different services:

- The NSI Services, as an implementation of the SDMX-RESTful web services specification, available (assuming you have a login) here:  https://webgate.ec.europa.eu/CITnet/stash/projects/SDMXRI/repos/nsiws.net/browse, and their associated data plugin, available here: https://gitlab.com/sis-cc/.stat-suite/dotstatsuite-core-sdmxri-nsi-plugin
- The Transfer service, which we'll use to load data and transfer it between spaces, available here: https://gitlab.com/sis-cc/.stat-suite/dotstatsuite-core-transfer
- The Authorization Management service, which we'll use to control access to data, available here: https://gitlab.com/sis-cc/.stat-suite/dotstatsuite-core-auth-management

We'll also be building (from source) and utilizing two different database tools:

- The MAAPI (Mapping Assistant API) database installation/upgrade tool, available (assuming you have a login) here: https://webgate.ec.europa.eu/CITnet/stash/projects/SDMXRI/repos/maapi.net/browse
- The DotStatSuite database installation/upgrade tool, available here: https://gitlab.com/sis-cc/.stat-suite/dotstatsuite-core-data-access

# The Topology

There are as many ways to set up your DotStatSuite suite as there are flavours of ice-cream, possibly more. There are a number of dimensions you can tweak when setting up your topology, but the main one is going to be how many dataspaces you want. In brief, you can think of each dataspace as a single set of SDMX structures and data, as stored in the databases we're going to be setting up. Each instance of the NSI services we install will belong to a single dataspace, whereas the Transfer and (to some degree) Authorization Management services sit across multiple dataspaces.

I'm going to be setting up a fairly common two-dataspace topology, representing the scenario where one space is used for design work, and the other is used for dissemination of statistics once they're ready for release. We'll be calling the two spaces design and disseminate respectively. For bonus points, we'll be creating two NSI service instances for the disseminate space; one internally-facing service that can upload SDMX structures, and another externally-facing service that cannot.

```
ASIDE

I'm going to be installing everything on one machine, but you can split up the services however makes the most sense for your requirements, so long as the required communication channels are open. For example, it's probably not a great idea to have your externally-facing disseminate-space NSI service on the same server as your internally-facing disseminate-space NSI service.
```

![TopologyDrawing](img/DotStatSuiteCore_Topology.png "Topology Diagram")

# Building from Source

## Prerequisites

In order to follow along and build all the various pieces of source code you're going to need a few things installed on your build machine (whether that's your personal machine, a build server, or a Docker container):
- .NET Core 3.1 SDK: The .NET Core 3.1 SDK is required to compile the .NET Core solutions we're working with. Not all solutions use version 3.1 (some at this writing are still at lower versions), but .NET Core SDKs can always build lower versions.
- Git: Required to clone the repositories. Obviously you could clone onto one machine and transfer to another machine to build.
- Powershell: You'll need to be able to run Powershell commands if you're following along. Obviously you don't **have** to use Powershell for any of this... any scripting language will be fine.

There are also some communication requirements. Wherever you're cloning the code to will need access to the Eurostat Bitbucket host (here: https://webgate.ec.europa.eu) and the Gitlab host (here: https://gitlab.com). I'm going to be cloning over HTTPS so I need access over port 443, but if you want to get fancy and clone over SSH, you'll need port 22.

Finally, unfortunately the repositories hosted on the Eurostat Bitbucket host are not open to anonymous access. You'll need a username and password with access to the following repositories:

- https://webgate.ec.europa.eu/CITnet/stash/projects/SDMXRI/repos/nsiws.net
- https://webgate.ec.europa.eu/CITnet/stash/projects/SDMXRI/repos/maapi.net

## Sourcing your Source

The first step towards building all the various services and utilities we need is to actually get all the sources downloaded onto our system. Below is the Powershell code snippet I'm using to do so:

```powershell
# Clone the db-up repository
& git clone -b $DataAccessRepoBranch --single-branch https://gitlab.com/sis-cc/.stat-suite/dotstatsuite-core-data-access.gitdotstatsuite-core-dbup
# Clone the maapi.net tool repo
& git clone -b $MaapiRepoBranch --single-branch --recurse-submodules https://${EurostatUserName}:$EurostatPassword@webgate.eceuropa.eu/CITnet/stash/scm/sdmxri/maapi.net.git
# Ensure that the authdb.sql.git submodule is cloned
& git clone -b $AuthDbRepoBranch --single-branch --recurse-submodules https://${EurostatUserName}:$EurostatPassword@webgate.eceuropa.eu/CITnet/stash/scm/sdmxri/authdb.sql.git maapi.net/src/Estat.Sri.Security/resources
# Clone the NSI web service repo
& git clone -b $NSIEurostatRepoBranch --single-branch https://${EurostatUserName}:$EurostatPassword@webgate.ec.europa.eu/CITnetstash/scm/sdmxri/nsiws.net.git
# Clone the NSI web service plugin repo
& git clone -b $NSIPluginRepoBranch --single-branch https://gitlab.com/sis-cc/.stat-suite/dotstatsuite-core-sdmxri-nsi-plugin.git
# Clone the Transfer service repo
& git clone -b $TransferRepoBranch --single-branch https://gitlab.com/sis-cc/.stat-suite/dotstatsuite-core-transfer.git
# Clone the AuthorizationManagement service repo
& git clone -b $AuthManagementRepoBranch --single-branch https://gitlab.com/sis-cc/.stat-suite/dotstatsuite-core-auth-management.git
```

