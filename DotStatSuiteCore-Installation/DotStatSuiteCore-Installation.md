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