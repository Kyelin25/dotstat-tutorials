# DRAFT

# Disclaimer

The purpose of this tutorial is to show an example of how to build and install the DotStatSuite .NET Core (also referred to as DotStatSuite Core) applications on a set of Windows servers. It is not intended as a guide on how to set it up securely, and taking any security advice unchecked from this would be exceptionally unwise. Use official documentation or advice from your organization's IT security team.

# Introduction

The purpose of this tutorial is to describe how one could go about building and deploying a simple working set of DotStatSuite Core services in a Windows environment. This means building on a Windows machine, and deploying **to** Windows machines. With this in mind, the scripting language I'll be using is Powershell, and the web server I'll be deploying on is Microsoft's Internet Information Services. However, a lot of the steps and information contained in this tutorial would be the same on other platforms, even if the exact commands might differ. Where possible I'll be noting what might be different.

This is the second part of the tutorial, and we'll be focusing on deploying the "packages" we built in the [first part](DotStatSuiteCore-Build.md).

# Prerequisites

This tutorial assumes that you have access to a database machine (or machines) and a web machine (or machines). In my case, I'm using one single computer as both a database machine and web machine, because I'm running everything locally.

The database machine needs:
- SQL Server installed. Ideally 2017 or later, but you may have luck with an earlier version.
- A SQL login created on the server with `sysadmin` rights. You'll need to know the username and password of this login.
- If you're following along with the tutorial exactly, the machine should be a 64-bit Windows machine, as our database packages were built with that in mind

The web machine needs:
- Internet Information Services installed
- You must have administrator permissions on the machine (relevant for running some Powershell scripts)
- The appropriate ASP.NET Core Hosting Bundle(s). At this writing of the tutorial that's the 3.1.3 Hosting Bundle and the 2.2.8 Hosting Bundle. The 2.2.8 Hosting Bundle will become unnecessary as soon as the last of the solutions moves to .NET Core 3.1.
- If you're using the code from this tutorial, you'll need the WebAdministration Powershell module installed. Most likely it is already installed by default.

# The Databases

We're going to be setting up our databases first (which makes sense... can't use the services without the databases!), but before we do that we'll have a brief runthrough of what databases we're going to end up initializing and for what they're used in the DotStat suite.

Each dataspace (see the [Topology](DotStatSuiteCore-Build.md#Topology) section of the previous tutorial for a **very** brief description of a dataspace) consists of three different databases:

- Data
- Structure (sometimes known as Mapping or MappingStore)
- Common

## Data Database

The Data database contains the information necessary for the proper functioning of the DotStat NSI plugin, including observation values and attribute values.

The scripts used to create the schema of the database are held [here](https://gitlab.com/sis-cc/.stat-suite/dotstatsuite-core-data-access/-/tree/develop/DotStat.DbUp.MsSql/DataDb).

## Structure Database

The Structure database (sometimes referred to as the Mapping or MappingStore database) is actually managed as part of the Eurostat codebase, which is why we need to use the MAAPI Database Tool to initialize it. It contains all the information necessary for the maintenance of SDMX structural metadata like Dataflows, DataStructures and Codelists.

## Common Database

The Common database contains the authorization rules that DotStat uses to determine if users are permitted to take specific actions. I'm not going to go into these in detail, but in essence each rule consists of a `scope`, a `user` or `group`, and a `permission`. When a user tries to do something on the system, it checks to see if there's a rule pertaining to them (or a group they're in), for the particular `scope` they're acting in that gives them sufficient `permission` to take that action. At this writing, only data access is currently managed through these rules.

One of the services we're going to install, the [Authorization Management Service](#Authorization-Management-Service), is used to manage the rules in this database.

The scripts used to create the schema of the database are held [here](https://gitlab.com/sis-cc/.stat-suite/dotstatsuite-core-data-access/-/tree/develop/DotStat.DbUp.MsSql/CommonDb).

# Naming Conventions

In this tutorial the code I'm using is pretty opinionated about what we call each database. For a given dataspace `{DataSpace}`, we create a `{DataSpace}CommonDb`, a `{DataSpace}DataDb` and a `{DataSpace}StructureDb`. However, there's no need for any particular naming convention. You **could** call them `One`, `Two` and `Three`, but it's not a bad idea to pick a naming convention that makes it clear what each database is for.

# Deploying the Databases

Alright, it's time to actually create and initialize our databases. We're going to start with the Data database, followed by the Common, then finally the most complicated: the Structure. As mentioned in the [Topology](DotStatSuiteCore-Build.md#Topology) section of the previous tutorial, we're going to have two dataspaces. That means of course we need two of each type of database.

## Data Database

The tool we use for the Data database is the DotStatSuite database installation tool, which we packaged in the subfolder `dbup`.

First, let's take a look at the Powershell function we're using to do this work:

```powershell
Function Install-DataDatabase{
    Param(
        [Parameter(Mandatory=$true)]
        [string]
        $DbUpInstallationPackageDirectory,

        [Parameter(Mandatory=$true)]
        [string]
        $DataspaceId,

        [Parameter(Mandatory=$true)]
        [string]
        $DatabaseServer,

        [Parameter(Mandatory=$true)]
        [string]
        $SysAdminUserName,

        [Parameter(Mandatory=$true)]
        [string]
        $SysAdminPassword,

        [Parameter(Mandatory=$true)]
        [string]
        $DatabaseUserToCreate,

        [Parameter(Mandatory=$true)]
        [string]
        $DatabaseUserToCreatePassword
    )

    if( -not ( Test-Path $DbUpInstallationPackageDirectory -PathType Container ) )
    {
        throw [System.IO.DirectoryNotFoundException]"Could not find DbUp installation package directory: $DbUpInstallationPackageDirectory"
    }

    # Work out the connection string we need to use. This means working out the database name too
    $DatabaseName="${DataspaceId}DataDb" # Opinionated naming convention for databases
    $DatabaseConnectionString="Server=${DatabaseServer};Database=${DatabaseName};User=${SysAdminUserName};Password=${SysAdminPassword};" # This could use Integrated Security instead of a sysadmin user

    Push-Location
    Set-Location $DbUpInstallationPackageDirectory

    # Create and initialise the database
    & .\DotStat.DbUp.exe upgrade --connectionString "$DatabaseConnectionString" --dataDb --loginName $DatabaseUserToCreate --loginPwd $DatabaseUserToCreatePassword --force

    Pop-Location
}
```

Here's how I'm calling the method, first for the `design` space, and secondly for the `disseminate` space (I've left):

```powershell
Install-DataDatabase -DbUpInstallationPackageDirectory C:\Development\OpenSource\Install\Packages\dotstatsuite-core-dbup -DataspaceId Design -DatabaseServer "localhost,1433" -SysAdminUserName sa -SysAdminPassword sysadm1n -DatabaseUserToCreate DesignDataLogin -DatabaseUserToCreatePassword DesignDataLog1n # Design space

Install-DataDatabase -DbUpInstallationPackageDirectory C:\Development\OpenSource\Install\Packages\dotstatsuite-core-dbup-DataspaceId Disseminate -DatabaseServer "localhost,1433" -SysAdminUserName sa -SysAdminPassword sysadm1n -DatabaseUserToCreate DisseminateDataLogin -DatabaseUserToCreatePassword DisseminateDataLog1n # Disseminate space
```

The line in the method that actually **creates** and then **initializes** the database, is this one:

```powershell
& .\DotStat.DbUp.exe upgrade --connectionString "$DatabaseConnectionString" --dataDb --loginName $DatabaseUserToCreate --loginPwd $DatabaseUserToCreatePassword --force
```

This line calls the tool we created in the previous tutorial, passing it the "verb" `upgrade` and providing it with the following arguments:

- connectionString: Pretty self-explanatory, this is the connection string it uses to perform actions against our database server. Of note though is that the "Database" section of the connection string will be the database the utility creates.
- dataDb: This flag tells the utility that we're creating a Data database (as opposed to a Common or Structure database).
- loginName and loginPwd: These two options correspond to the SQL login (and user) that the utility will **create**. This is the login that applications will need to use in order to connect to the newly-created database.
- force: This flag runs the utility in non-interactive mode, so we don't get prompted for permission to do things.

It's worth noting that the verb `upgrade` is also what we would use to take an existing database and upgrade it to a newer version of the DotStat suite, without damaging what we already have in it (that said, it would be very foolish to ever upgrade a production database without a restoration strategy in case something goes wrong). The other verbs can be listed by calling the executable with no arguments at all (`DotStat.DbUp.exe`), and will be explored later in an aside at the end of the database section of this tutorial.

## Common Database

The tool we use for the Common database is the DotStatSuite database installation tool, which we packaged in the subfolder `dbup`, just the same as we used for the Data database.

First, let's take a look at the Powershell function we're using to do this work:

```powershell
Function Install-CommonDatabase{
    Param(
        [Parameter(Mandatory=$true)]
        [string]
        $DbUpInstallationPackageDirectory,

        [Parameter(Mandatory=$true)]
        [string]
        $DataspaceId,

        [Parameter(Mandatory=$true)]
        [string]
        $DatabaseServer,

        [Parameter(Mandatory=$true)]
        [string]
        $SysAdminUserName,

        [Parameter(Mandatory=$true)]
        [string]
        $SysAdminPassword,

        [Parameter(Mandatory=$true)]
        [string]
        $DatabaseUserToCreate,

        [Parameter(Mandatory=$true)]
        [string]
        $DatabaseUserToCreatePassword
    )

    if( -not ( Test-Path $DbUpInstallationPackageDirectory -PathType Container ) )
    {
        throw [System.IO.DirectoryNotFoundException]"Could not find DbUp installation package directory: $DbUpInstallationPackageDirectory"
    }

    # Work out the connection string we need to use. This means working out the database name too
    $DatabaseName="${DataspaceId}CommonDb" # Opinionated naming convention for databases
    $DatabaseConnectionString="Server=${DatabaseServer};Database=${DatabaseName};User=${SysAdminUserName};Password=${SysAdminPassword};"  # This could use Integrated Security instead of a sysadmin user

    Push-Location
    Set-Location $DbUpInstallationPackageDirectory

    # Create and initialise the database
    & .\DotStat.DbUp.exe upgrade --connectionString "$DatabaseConnectionString" --commonDb --loginName $DatabaseUserToCreate --loginPwd $DatabaseUserToCreatePassword --force

    Pop-Location
}
```

Here's how I'm calling the method, first for the `design` space, and secondly for the `disseminate` space (I've left):

```powershell
Install-CommonDatabase -DbUpInstallationPackageDirectory C:\Development\OpenSource\Install\Packages\dotstatsuite-core-dbup -DataspaceId Design -DatabaseServer "localhost,1433" -SysAdminUserName sa -SysAdminPassword sysadm1n -DatabaseUserToCreate DesignCommonLogin -DatabaseUserToCreatePassword DesignCommonLog1n # Design space

Install-CommonDatabase -DbUpInstallationPackageDirectory C:\Development\OpenSource\Install\Packages\dotstatsuite-core-dbup -DataspaceId Disseminate -DatabaseServer "localhost,1433" -SysAdminUserName sa -SysAdminPassword sysadm1n -DatabaseUserToCreate DisseminateCommonLogin -DatabaseUserToCreatePassword DisseminateCommonLog1n # Disseminate space
```

The line in the method that actually **creates** and then **initializes** the database, is this one:

```powershell
& .\DotStat.DbUp.exe upgrade --connectionString "$DatabaseConnectionString" --commonDb --loginName $DatabaseUserToCreate --loginPwd $DatabaseUserToCreatePassword --force
```

This line calls the tool we created in the previous tutorial, passing it the "verb" `upgrade` and providing it with the following arguments:

- connectionString: Pretty self-explanatory, this is the connection string it uses to perform actions against our database server. Of note though is that the "Database" section of the connection string will be the database the utility creates.
- commonDb: This flag tells the utility that we're creating a Common database (as opposed to a Data or Structure database).
- loginName and loginPwd: These two options correspond to the SQL login (and user) that the utility will **create**. This is the login that applications will need to use in order to connect to the newly-created database.
- force: This flag runs the utility in non-interactive mode, so we don't get prompted for permission to do things.

It's worth noting that the verb `upgrade` is also what we would use to take an existing database and upgrade it to a newer version of the DotStat suite, without damaging what we already have in it (that said, it would be very foolish to ever upgrade a production database without a restoration strategy in case something goes wrong). The other verbs can be listed by calling the executable with no arguments at all (`DotStat.DbUp.exe`), and will be explored later in an aside at the end of the database section of this tutorial.

## Structure Database

This is where things get ever so slightly hairier, because we're going to be using **two** database tools: the DotStat database installation tool, which we packaged in the subfolder `dbup`, and the MAAPI database installation tool, which we packaged in the subfolder `maapi`.

The reason we're using two tools, is because the MAAPI tool will only run the scripts to initialize the database. It won't create the database, nor will it set up the database user we need, so we need to get the DotStat tool first.

Further complicating things is that the MAAPI tool doesn't accept the connection string as an argument. Instead, it looks for it in a configuration file `Estat.Sri.Mapping.Tool.dll.config`. Now, you can just manually edit that file to add your connection string, but I felt like it would be easier to edit it automatically in our deployment function. Speaking of which, here's the Powershell function we're using in its entirety:

```powershell
Function Install-StructureDatabase{
    Param(
        [Parameter(Mandatory=$true)]
        [string]
        $MaapiInstallationPackageDirectory,

        [Parameter(Mandatory=$true)]
        [string]
        $DbUpInstallationPackageDirectory,

        [Parameter(Mandatory=$true)]
        [string]
        $DataspaceId,

        [Parameter(Mandatory=$true)]
        [string]
        $DatabaseServer,

        [Parameter(Mandatory=$true)]
        [string]
        $SysAdminUserName,

        [Parameter(Mandatory=$true)]
        [string]
        $SysAdminPassword,

        [Parameter(Mandatory=$true)]
        [string]
        $DatabaseUserToCreate,

        [Parameter(Mandatory=$true)]
        [string]
        $DatabaseUserToCreatePassword
    )

    if( -not ( Test-Path $MaapiInstallationPackageDirectory -PathType Container ) )
    {
        throw [System.IO.DirectoryNotFoundException]"Could not find maapi installation package directory: $MaapiInstallationPackageDirectory"
    }

    if( -not ( Test-Path $DbUpInstallationPackageDirectory -PathType Container ) )
    {
        throw [System.IO.DirectoryNotFoundException]"Could not find DbUp installation package directory: $DbUpInstallationPackageDirectory"
    }

    # Work out the connection string we need to use. This means working out the database name too
    $DatabaseName="${DataspaceId}StructDb" # Opinionated naming convention for databases
    $DatabaseConnectionString="Server=${DatabaseServer};Database=${DatabaseName};User=${SysAdminUserName};Password=${SysAdminPassword};"  # This could use Integrated Security instead of a sysadmin user

    Push-Location
    Set-Location $MaapiInstallationPackageDirectory

    # Calculate the path of the configuration file
    $ToolConfigurationFile=Join-Path $MaapiInstallationPackageDirectory Estat.Sri.Mapping.Tool.dll.config

    # Retrieve the configuration file content
    [xml]$ConfigFileContent=Get-Content -Path $ToolConfigurationFile

    # Update the sqlserver connection string to the one we want
    $ConfigFileContent.SelectSingleNode("/configuration/connectionStrings/add[@name='sqlserver']").connectionString=$DatabaseConnectionString

    # Write it back
    $ConfigFileContent.Save($ToolConfigurationFile)

    # Now that we've set up maapi, let's use DbUp to create the database
    Set-Location $DbUpInstallationPackageDirectory

    & .\DotStat.DbUp.exe upgrade --connectionString "$DatabaseConnectionString" --mappingStoreDb --loginName $DatabaseUserToCreate --loginPwd $DatabaseUserToCreatePassword --force

    # The database now exists. Time to turn it into a mapping store db
    Set-Location $MaapiInstallationPackageDirectory

    & .\Estat.Sri.Mapping.Tool.exe init -m sqlserver -f

    Pop-Location
}
```

And here's us calling it to create the two spaces, first `design` and then `disseminate`:

```powershell
Install-StructureDatabase -MaapiInstallationPackageDirectory C:\Development\OpenSource\Install\Packages\maapi -DbUpInstallationPackageDirectory C:\Development\OpenSource\Install\Packages\dotstatsuite-core-dbup -DataspaceId Design -DatabaseServer "localhost,1433" -SysAdminUserName sa -SysAdminPassword sysadm1n -DatabaseUserToCreate DesignStructureLogin -DatabaseUserToCreatePassword DesignStructureLog1n # Design space

Install-StructureDatabase -MaapiInstallationPackageDirectory C:\Development\OpenSource\Install\Packages\maapi -DbUpInstallationPackageDirectory C:\Development\OpenSource\Install\Packages\dotstatsuite-core-dbup -DataspaceId Disseminate -DatabaseServer "localhost,1433" -SysAdminUserName sa -SysAdminPassword sysadm1n -DatabaseUserToCreate DisseminateStructureLogin -DatabaseUserToCreatePassword DisseminateStructureLog1n # Disseminate space
```

### The MAAPI Connection String

Okay, so before we can get to the calls to the tools, we should cover off how we're setting the connection string for the MAAPI tool. The configuration file `Estat.Sri.Mapping.Tool.dll.config` is read into the application when you execute it and has a lot of configuration settings that you really don't need to mess about with. What it also contains is a set of named connectionStrings. When you execute the application, one of the arguments you pass is the name of the connectionString you want it to use. Because we're using SQL Server, we need to use the `sqlserver`, and we can ignore what the others say, because we're not using them. The code we use to replace it simply reads the contents of the file as XML, then uses an XPath expression to find the `sqlserver` connectionString and replace it, before writing the contents back to the file.

### Creating the Database

The line of code that actually creates the database (and an application user) is the following:

```powershell
& .\DotStat.DbUp.exe upgrade --connectionString "$DatabaseConnectionString" --mappingStoreDb --loginName $DatabaseUserToCreate --loginPwd $DatabaseUserToCreatePassword --force
```

This should look pretty familiar by now. We're calling the the DotStat database installation tool with the "verb" `upgrade` and passing it the following arguments:

- connectionString: Pretty self-explanatory, this is the connection string it uses to perform actions against our database server. Of note though is that the "Database" section of the connection string will be the database the utility creates.
- mappingStoreDb: This flag tells the utility that we're creating a Structure database (as opposed to a Data or Common database). Remember, that the Structure database is often referred to as the MappingStore database.
- loginName and loginPwd: These two options correspond to the SQL login (and user) that the utility will **create**. This is the login that applications will need to use in order to connect to the newly-created database.
- force: This flag runs the utility in non-interactive mode, so we don't get prompted for permission to do things.

Unlike the previous times we called this tool, it doesn't populate the database with what it needs to function. This work falls to the MAAPI tool.

### Initializing the Database

Once we've used the DotStat database tool to create the database and our application user, we need to actually populate the Structure database with all the database objects it needs to be used **as** a Structure database. This is the line where we do that:

```powershell
& .\Estat.Sri.Mapping.Tool.exe init -m sqlserver -f
```

In this code, we invoke the MAAPI executable with the "verb" `init` and pass it two arguments:

- m: This corresponds to which of the connectionStrings in `Estat.Sri.Mapping.Tool.dll.config` we want to use, so as discussed, we're using `sqlserver`.
- f: The force flag simply avoids us getting prompted

It's worth noting that the there is also a verb `upgrade` which is what we would use to take an existing Structure database and upgrade it to a newer version, without damaging what we already have in it (that said, it would be very foolish to ever upgrade a production database without a restoration strategy in case something goes wrong). The one we used, `init`, will initialize a new database, and if you run it against an existing one, it will **remove** all the previous initialization, so be careful!

The other verbs can be listed by calling the executable with no arguments at all (`Estat.Sri.Mapping.Tool.exe`), and will be explored later in an aside at the end of the database section of this tutorial.

## Aside - Tool Use

By this stage, we've successfully created and initialized all our databases, and we're ready to move on to deploying the services themselves. So, if you don't want to read more about other uses of the database tools, feel free to skip this section.

As alluded to in the previous sections, there are more ways to use the tools, the most important of which is using them to upgrade from an older database version to a newer one. You can list the available verbs either by simply executing the tool with no arguments (causing an error that spits out the available verbs) or using the `help` verb. Executing the `help` verb followed by another verb will provide further information about its use.

First, let's list all the verbs and what they do. The MAAPI verbs are:

- list: This claims to list available mapping stores. What it will do is loop over all the connectionStrings in `Estat.Sri.Mapping.Tool.config` and determine if it is an initialized MappingStore (read as Structure database) and what database version it is. It should also then list available versions, but I can't see that anywhere. In our config case, you'll get errors because two of the connectionStrings are trash.
- init: We already used this one. This will initialize a new Structure database. If run against an existing one, it'll wipe it out before doing so. Use the `-m` argument to select which connectionString to use, and the `-f` argument to avoid prompts.
- upgrade: This verb can be used to upgrade an existing Structure database to a newer version. Use the `-m` argument to select which connectionString to use, and the `-f` argument to avoid prompts. The `-t` argument specifies the target database version (e.g. `-t 6.8`), but claims to only support the latest version, which implies to me that it doesn't actually do anything. Instead, using this verb will upgrade to the latest version available to the tool.

The DotStat verbs are:

- executed: This verb will list which database initialization scripts have already been run on an existing database. Use the `--connectionString` argument to specify the connection string to connect to the database, and one of `--commonDb`, `--dataDb` and `--mappingStoreDb` to specify whether you want the information for a Common, Data or Structure database. Keep in mind that for the Structure database, you'll only get information about the scripts **this** tool has run, **not** any scripts run by the MAAPI tool.
- tobeexecuted: This verb will list which database initialization scripts **would** be run on a database if the `upgrade` verb were to be used. This is incredibly useful for generated a pre-flight report you can check before committing to upgrading a database. Use the `--connectionString` argument to specify the connection string to connect to the database, and one of `--commonDb`, `--dataDb` and `--mappingStoreDb` to specify whether you want the information for a Common, Data or Structure database. Keep in mind that for the Structure database, you'll only get information about the scripts **this** tool will run, **not** any scripts that would be run by the MAAPI tool.