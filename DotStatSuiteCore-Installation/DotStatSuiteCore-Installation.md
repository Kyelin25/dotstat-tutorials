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
& git clone -b $DataAccessRepoBranch --single-branch https://gitlab.com/sis-cc/.stat-suite/dotstatsuite-core-data-access.git dotstatsuite-core-dbup

# Clone the maapi.net tool repo (along with its submodules)
& git clone -b $MaapiRepoBranch --single-branch --recurse-submodules https://${EurostatUserName}:$EurostatPassword@webgate.ec.europa.eu/CITnet/stash/scm/sdmxri/maapi.net.git

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
git clone -b release3.0.0 --single-branch https://gitlab.com/sis-cc/.stat-suite/dotstatsuite-core-data-access.git dotstatsuite-core-dbup

git clone -b 1.25.1 --single-branch --recurse-submodules https://ben:password@webgate.ec.europa.eu/CITnet/stash/scm/sdmxri/maapi.net.git

git clone -b 7.11.1 --single-branch https://ben:password@webgate.ec.europa.eu/CITnetstash/scm/sdmxri/nsiws.net.git

git clone -b 7.11.1 --single-branch https://gitlab.com/sis-cc/.stat-suite/dotstatsuite-core-sdmxri-nsi-plugin.git

git clone -b release3.0.0 --single-branch https://gitlab.com/sis-cc/.stat-suite/dotstatsuite-core-transfer.git

git clone -b release3.0.0 --single-branch https://gitlab.com/sis-cc/.stat-suite/dotstatsuite-core-auth-management.git
```

Now we have our source repositories cloned, with the specific branches we're looking for, and they're sitting in the following subfolders:

- DotStatSuite Db Tool: dotstatsuite-core-dbup
- MAAPI Db Tool: maapi.net
- NSI Service: nsiws.net
- NSI Plugin: dotstatsuite-core-sdmxri-nsi-plugin
- Transfer Service: dotstatsuite-core-transfer
- Authorization Management Service: dotstatsuite-core-auth-management

The next step is to build them and create some installation packages.

# Building our Packages

Building and packaging most of the solutions we need is as simple as a single `dotnet` command (with the very notable exception of the NSI services), but we'll go through each of them in some detail anyway. Feel free to skim over the really obvious ones. This order is completely arbitrary.

## The Simple Services

Building the Transfer and Authorization Management services is almost trivial, so we'll start with them.

### Transfer Service

Publishing the Transfer service is as simple as can be. Here's the Powershell method we're using:

```powershell
Function Publish-TransferService{
    Param(
        [Parameter(Mandatory=$true)]
        [string]
        $RepositoryDirectory,

        [Parameter(Mandatory=$true)]
        [string]
        $PackageOutputDirectory,

        [string]
        $Version="Release"
    )

    if( -not (Test-Path $RepositoryDirectory -PathType Container ) ){
        throw [System.IO.DirectoryNotFoundException]"Could not find repository directory: $RepositoryDirectory"
    }

    Push-Location
    Set-Location $RepositoryDirectory

    # Build the Transfer service
    & dotnet publish -c $Version -o $PackageOutputDirectory

    Pop-Location
}
```

This is equivalent to executing the following simple statement from our root folder:

```sh
dotnet publish ./dotstatsuite-core-transfer -c Release -o ./packages/transfer
```

The output of this is a folder filled with (almost) everything we need to deploy the Transfer service. The "almost" is because of configuration we'll need to do, and we'll cover that when it's time to deploy. The point is, this package should be able to be deployed to anywhere we want to host it.

#### Of Note

There's almost nothing of note here, except that we're choosing to use the Release configuration of the service, as this will lead to better optimized code, among other things. We'll be doing this for all the services. The default is `Debug`, which can lead to you deploying non-optimized code (but a lot easier to debug!).

### Authorization Management Service

Publishing the Authorization Management service is just as easy as publishing the Transfer service. Here's the Powershell method we're using:

```powershell
Function Publish-AuthManagementService{
    Param(
        [Parameter(Mandatory=$true)]
        [string]
        $RepositoryDirectory,

        [Parameter(Mandatory=$true)]
        [string]
        $PackageOutputDirectory,

        [string]
        $Version="Release"
    )

    if( -not (Test-Path $RepositoryDirectory -PathType Container ) ){
        throw [System.IO.DirectoryNotFoundException]"Could not find repository directory: $RepositoryDirectory"
    }

    Push-Location
    Set-Location $RepositoryDirectory

    # Build the Authorization Management service
    & dotnet publish -c $Version -o $PackageOutputDirectory

    Pop-Location
}
```

This is equivalent to executing the following simple statement from our root folder:

```sh
dotnet publish ./dotstatsuite-core-auth-management -c Release -o ./packages/authmanagement
```

The output of this is a folder filled with (almost) everything we need to deploy the Authorization Management service. The "almost" is because of configuration we'll need to do, and we'll cover that when it's time to deploy. The point is, this package should be able to be deployed to anywhere we want to host it.

#### Of Note

There's almost nothing of note here, except that we're choosing to use the Release configuration of the service, as this will lead to better optimized code, among other things. We'll be doing this for all the services. The default is `Debug`, which can lead to you deploying non-optimized code (but a lot easier to debug!).

## The Database Tools

These are the first ever so slightly curly ones. I'm installing everything (all the services, all the databases) on one machine, so I can be certain that where I run the database tools will have the .NET Core runtime (because otherwise I couldn't host the services!). However, in general, your databases will be hosted on a separate server, and that server probably shouldn't have things installed on it that it doesn't need; such as a .NET Core runtime. So, what we're going to do is publish the database tools as [Self-contained deployments](https://docs.microsoft.com/en-us/dotnet/core/deploying/deploy-with-cli#self-contained-deployment). Over-simplifying, we're going to publish them as exe files that we can run on the database server without needing a .NET Core runtime. This is as easy as adding an option to the `dotnet publish` command.

### DotStatSuite Db Tool

Aside from wanting to produce an executable, this step is not much more complicated than that for the Transfer or Authorization Management services. Here's the Powershell method we're using:

```powershell
Function Publish-DbUp{
    Param(
        [Parameter(Mandatory=$true)]
        [string]
        $RepositoryDirectory,

        [Parameter(Mandatory=$true)]
        [string]
        $PackageOutputDirectory,

        [string]
        $Version="Release"
    )

    if( -not (Test-Path $RepositoryDirectory -PathType Container ) ){
        throw [System.IO.DirectoryNotFoundException]"Could not find repository directory: $RepositoryDirectory"
    }

    Push-Location
    Set-Location $RepositoryDirectory

    # Build the dotstatsuite-core-dbup tool, using win-x64 as the Runtime Identifier (to run on 64-bit windows machines)
    & dotnet publish ./DotStat.DbUp -c Release -o $PackageOutputDirectory -r win-x64

    Pop-Location
}
```

This is equivalent to running the following simple statement from our root folder (no promises that you can build a win-x64 executable from Linux... let me know how that goes):

```sh
dotnet publish ./dotstatsuite-db-up/DotStat.DbUp -c Release -o ./packages/dbup -r win-x64
```

The output of this is a folder containing an exe file (`DotStat.DbUp.exe`), all the files **it** needs, as well as a copy of the entire .NET Core runtime, meaning you can run this on any appropriate 64-bit Windows machine, without having to install the .NET Core runtime.

#### Of Note

Obviously, the use of the Runtime Identifier option in the `dotnet publish` command is notable. It's not necessary of course if you're planning to do what I'm doing (installing everything on one server), or plan to run the database tooling from a .NET Core enabled server that can connect to your SQL Server instance.

Also of note, is that for the first time, we're targetting a subfolder of the repository with our `dotnet publish` command. This is simply because the [dotstatsuite-core-data-access](https://gitlab.com/sis-cc/.stat-suite/dotstatsuite-core-data-access) repository contains not only the database tool, but also common libraries used by other DotStatSuite services. The database tool sits in the DotStat.DbUp subfolder (so named as it uses the [DbUp](https://dbup.readthedocs.io/en/latest/) deployment library).

### MAAPI Db Tool

Publishing the MAAPI Db tool is essentially the same as publishing the DotStatSuite Db tool. Here's the Powershell method we're using:

```powershell
Function Publish-MaapiDbTool{
    Param(
        [Parameter(Mandatory=$true)]
        [string]
        $RepositoryDirectory,

        [Parameter(Mandatory=$true)]
        [string]
        $PackageOutputDirectory,

        [string]
        $Version="Release"
    )

    if( -not (Test-Path $RepositoryDirectory -PathType Container ) ){
        throw [System.IO.DirectoryNotFoundException]"Could not find repository directory: $RepositoryDirectory"
    }

    Push-Location
    Set-Location $RepositoryDirectory

    # Build the maapi.net tool, using win-x64 as the Runtime Identifier (to run on 64-bit windows machines)
    & dotnet publish ./src/Estat.Sri.Mapping.Tool/Estat.Sri.Mapping.Tool.csproj -c $Version -o $PackageOutputDirectory -r win-x64

    Pop-Location
}
```

This is equivalent to running the following simple statement from our root folder (no promises that you can build a win-x64 executable from Linux... let me know how that goes):

```sh
dotnet publish ./maapi.net/Estat.Sri.Mapping.Tool/Estat.Sri.Mapping.Tool.csproj -c Release -o ./packages/maapi -r win-x64
```

The output of this is a folder containing an exe file (`Estat.Sri.Mapping.Tool.exe`), all the files **it** needs, as well as a copy of the entire .NET Core runtime, meaning you can run this on any appropriate 64-bit Windows machine, without having to install the .NET Core runtime.

#### Of Note

Obviously, the use of the Runtime Identifier option in the `dotnet publish` command is notable. It's not necessary of course if you're planning to do what I'm doing (installing everything on one server), or plan to run the database tooling from a .NET Core enabled server that can connect to your SQL Server instance.

Also of note, is that just as we did for the DotStatSuite Db tool, we're targetting a subfolder of the repository with our `dotnet publish` command. This is for the same reason. The [maapi.net](https://webgate.ec.europa.eu/CITnet/stash/projects/SDMXRI/repos/maapi.net) repository contains source code for other shared libraries and not just the tool. The database tool sits in the Estat.Sri.Mapping.Tool subfolder.

## The NSI Services - Some Assembly Required

This is where things get curly. The DotStat suite uses the NSI web services as its SDMX-RESTful web service, but provides its own plugin to the service to manage the retrieval of data (structure is done using the defaults provided in the NSI services). To us, what that means is that we need to not only build the NSI service and the plugin, but perform some assembly to put them together.

### NSI Service

First let's take a quick look at the entire Powershell method we're using to produce the NSI package. Then, we'll break it down step by step and explain why we're doing what we're doing.

```powershell

```