# Purpose

 The purpose of this document is to provide an overview of the solutions that make up the overall .Stat Suite, along with how the solutions fit together. It will focus heavily on information that would be useful for a developer or integration analyst setting up the .Stat Suite, and where possible will be technology-agnostic.

 # Some Background

 ## The .NET and JavaScript Split

 The .Stat Suite solutions can be divided into two groups:
 - The .NET Components, also known as the Core components, are all written in .NET Core, and generally consist of web APIs for managing and retrieving the contents of an SDMX datasource
 - The JavaScript Components are all written in JavaScript (NodeJS and ReactJS) and generally have the purpose of supporting end-user interactions

 This split is not only important in terms of what infrastructure is required to run the groups, but because it is entirely possible to install the .NET Components without the JavaScript Components and vice versa (though as you'll see in the section on, this isn't the full story).

 Thus, we'll be splitting the descriptions of the .Stat Suite components along these lines.

 ## Tenants

 The concept of a *tenant* is something you'll need to keep in mind when deploying and configuring the JavaScript components. Conceptually tenants represent organisations or groups sharing instances of the JavaScript solutions, but using their own themes and configurations. This extends even to sourcing data from different *datasources*.

 From a deployment and configuration point of view though, a tenant is just a collection of configuration settings (not all settings can differ between tenants, and not all JavaScript solutions are tenant-aware {Nicolas-Review}). The key thing you'll need to consider when deploying and configuring the JavaScript components is the following question:

 ```
 Do I need multiple tenants (sets of configurations) or do I only need the one?
 ```

