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

- The MAAPI (Mapping Assistant API) database installation/upgrade tool, available (assuming you have a login) here: https://webgate.ec.europa.eu/CITnet/stash/projects/SDMXRI/repos/maapi.net/browse, which has a submodule hosted here: https://webgate.eceuropa.eu/CITnet/stash/scm/sdmxri/authdb.sql.git
- The DotStatSuite database installation/upgrade tool, available here: https://gitlab.com/sis-cc/.stat-suite/dotstatsuite-core-data-access

# The Topology

There are as many ways to set up your DotStatSuite suite as there are flavours of ice-cream, possibly more. There are a number of dimensions you can tweak when setting up your topology, but the main one is going to be how many dataspaces you want. In brief, you can think of each dataspace as a single set of SDMX structures and data, as stored in the databases we're going to be setting up. Each instance of the NSI services we install will belong to a single dataspace, whereas the Transfer and (to some degree) Authorization Management services sit across multiple dataspaces.

I'm going to be setting up a fairly common two-dataspace topology, representing the scenario where one space is used for design work, and the other is used for dissemination of statistics once they're ready for release. We'll be calling the two spaces design and disseminate respectively. For bonus points, we'll be creating two NSI service instances for the disseminate space; one internally-facing service that can upload SDMX structures, and another externally-facing service that cannot.

```
ASIDE

I'm going to be installing everything on one machine, but you can split up the services however makes the most sense for your requirements, so long as the required communication channels are open. For example, it's probably not a great idea to have your externally-facing disseminate-space NSI service on the same server as your internally-facing disseminate-space NSI service.
```

![TopologyDrawing](img/DotStatSuiteCore_Topology.png "Topology Diagram")

# Versions

The DotStatSuite Core suite of services consists of a number of services, which all have their own versions. Furthermore, all of them rely on a set of databases, which are also versioned. To make things even more complicated, the NSI services and the structure database are not versioned by the same group as the Transfer and Authorization Management services and the data database. This can make working out what branches or tags to build a bit of a nightmare. However, not all is doom and gloom. It **is** possible to determine what version to use.

Firstly, the parts of the suite hosted on Gitlab have *releases* detailed [here](https://sis-cc.gitlab.io/dotstatsuite-documentation/changelog/) on the ".Stat Suite documentation" website. Furthermore, this generally means that each of the repositories involved in a release will have a release tag corresponding **to** that release. For example, I'm building the .NET 3.2.0 release, so I'll be looking for tags like "release3.2.0". At this stage unfortunately if a repository doesn't take part in a release (because that service hasn't changed) it doesn't necessarily receive the release tag. In that case your best bet is probably to go to the next latest release tag.

Secondly, generally in the release description (again, found [here](https://sis-cc.gitlab.io/dotstatsuite-documentation/changelog/)) it'll mention which version of the NSI services it's using. If not, your next step is to go to the appropriate branch/tag of the [plugin repository](https://gitlab.com/sis-cc/.stat-suite/dotstatsuite-core-sdmxri-nsi-plugin) (see above for how to work **that** out) and have a look at the CHANGELOG.md file, which should have the NSI version history of the plugin. You'll take the latest one, of course. For example, I'm building the .NET 3.2.0 so I'd look [here](https://gitlab.com/sis-cc/.stat-suite/dotstatsuite-core-sdmxri-nsi-plugin/-/blob/release3.2.0/CHANGELOG.md) and see that the latest version of the NSI services is 7.11.1.

Finally you need to work out what versions of the database tools you need. This is fairly easy for the DotStatSuite database tool (hosted [here](https://gitlab.com/sis-cc/.stat-suite/dotstatsuite-core-data-access)) as it follows the same *release* tag pattern as the rest of the Gitlab repositories. From there in order to work out what version of the MAAPI database tool you need, you can either go check the changelog, or the DotStat.MappingStore.csproj reference.

 If looking at the changelog, I (using .NET 3.2.0) would go [here](https://gitlab.com/sis-cc/.stat-suite/dotstatsuite-core-data-access/-/blob/release3.2.0/CHANGELOG.md) and see which version of the MSDB we're on (6.8 in this case). Then, I would go to the [maapi.net changelog](https://webgate.ec.europa.eu/CITnet/stash/projects/SDMXRI/repos/maapi.net/browse/CHANGELOG.md) and find the latest repository version for that MSDB version (in my case, 1.25.2 as 1.25.3 uses MSDB version 6.9).

 If looking at the csproj reference, I would go to the [DotStat.MappingStore.csproj file](https://gitlab.com/sis-cc/.stat-suite/dotstatsuite-core-data-access/-/blob/release3.2.0/DotStat.MappingStore/DotStat.MappingStore.csproj) for my release, and simply take whatever version of the `Estat.Sri.Sdmx.MappingStore.Store` package is in use (in my case, 1.25.1). Either way I'll end up with the correct database versions.

 By following these steps, I've decided I'm going to be pulling the following branches/tags:
 - NSI Services: 7.11.1
 - NSI Plugin: release3.2.0
 - Transfer Service: release3.0.0 (as it did not take part in release3.2.0)
 - Authorization Management Service: release3.0.0 (as it did not take part in release3.2.0)
 - DotStatSuite Db Tool: release3.2.0
 - MAAPI Db Tool: 1.25.1

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
- https://webgate.eceuropa.eu/CITnet/stash/scm/sdmxri/authdb.sql.git (this is a submodule of the maapi.net repository)

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

The first thing you'll notice is that it's heavily parameterized. This is because it's part of a set of Powershell functions I've set up to make producing and installing DotStatSuite Core packages easier (also, I'd really rather not post my Eurostat password here). The second thing is probably the weird `&` symbol at the front of each of the commands. This is because I'm calling the Git executable from Powershell. Here are the same commands with the parameters filled in with example values, as you might call them in the terminal on a Linux machine:

```sh
git clone -b release3.0.0 --single-branch https://gitlab.com/sis-cc/.stat-suite/dotstatsuite-core-data-access.gitdotstatsuite-core-dbup

git clone -b 1.25.1 --single-branch --recurse-submodules https://ben:password@webgate.eceuropa.eu/CITnet/stash/scm/sdmxri/maapi.net.git

git clone -b 1.0.1 --single-branch --recurse-submodules https://ben:password@webgate.eceuropa.eu/CITnet/stash/scm/sdmxri/authdb.sql.git maapi.net/src/Estat.Sri.Security/resources

git clone -b 7.11.1 --single-branch https://ben:password@webgate.ec.europa.eu/CITnetstash/scm/sdmxri/nsiws.net.git

git clone -b 7.11.1 --single-branch https://gitlab.com/sis-cc/.stat-suite/dotstatsuite-core-sdmxri-nsi-plugin.git

git clone -b release3.0.0 --single-branch https://gitlab.com/sis-cc/.stat-suite/dotstatsuite-core-transfer.git

git clone -b release3.0.0 --single-branch https://gitlab.com/sis-cc/.stat-suite/dotstatsuite-core-auth-management.git
```