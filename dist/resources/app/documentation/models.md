# Updating Orb Models

* [Back to Help](all.md)
* [PowerShell Objects](powershellObjects.md)

>* You can Add/Remove/Update Objects and Namespaces in the Edit Page.

>* The fastest way to create a new namespace is to browse through existing examples and clone one.

>* To edit existing objects, you can also right-click on any Object in the Explorer Page and click Edit.

# Models Overview

Orb uses a git repository to store all object and resource definitions.

>* Orb automatically syncs with the Models git repository to continuously update definitions.

The Orb Models Repository is located [here.](https://dev.azure.com/orbModels/_git/OrbModels)

You can explore your local git repository in the Edit page. You can also right-click folders and click 'Reveal in Explorer'.

>* Any changes made locally need to be explicitly pushed through Pull Requests in the *Changes* section.

## Public Models

Anything located under src\Models is considered Public. Public Model updates require no Code Reviews.

## Protected Models

Anything located under src\ProtectedModels is considered Protected. Protected Models require Code Reviews.

>* All PowerShell Object definitions and resources (psx, psmd, terminal, etc.) must be placed under the Protected Models folder.
>* Since Constants can be used by any definition, these have to be added to Protected Models.

# Namespaces

>* Namespaces are designed to be at the scope of a large organization and not individual teams.

Object trees can span namespaces if required.

## Creating a new Namespace

To create a namespace, you need to create a new folder and a namespaceConfig.json file.

You can pick one of the 3 directory layouts below.

### Fully Public Definition

>* This setup is recommended if you plan to use Kusto objects and non-PowerShell resources.

For example, to create a namespace "foo", this should be the folder structure.

<pre>
Models
|
foo
   |
   namespaceConfig.json
   |
   Objects
      |
      object1.json
</pre>

### Fully Protected Definition

If you want your entire namespace config to be only modifiable through code reviews, you can place it in the protected folder.

>* You can create owners.txt files at every folder to control Ownership Enforcer approvals.
>* This setup is recommended if you plan to only use PowerShell objects and resources.

<pre>
ProtectedModels
|
foo
   |
   owners.txt
   namespaceConfig.json
   |
   Objects
      |
      object1.json
</pre>

### Hybrid Definition (Most Flexible)

You can choose to split your config into multiple files and decide what portion goes through code reviews. Note that PowerShell related definition must always go in the protected folder.

>* This setup is recommended if you plan to use a combination of Kusto and PowerShell objects/resources.

<pre>
src
|
Models
 |
 foo
   |
   namespaceConfig.json
   |
   Objects
      |
      object1Hybrid.json
      object2FullyPublic.json
|
ProtectedModels
 |
 foo
   |
   owners.txt
   namespaceconfig.protected.json #Contains just the stuff that needs CRs
   |
   Objects
      |
      owners.txt
      object1Hybrid.protected.json #Contains just the stuff that needs CRs
      object3FullyProtected.json
</pre>


## NamespaceConfig.json

The namespaceConfig contains several resource profiles.

These resource profiles tell Orb about how to connect to key resources like Kusto, Dgrep, etc.

This allows individual resource definitions to share connection configuration.

Sample namespace config for OrbSample:

```json
{
    "name": "OrbSample",
    "requiredBaseProps": [
        {
            "name": "cloudType",
            "type": "enum",
            "label": "Environment",
            "value": [
                "Public",
                "Private",
            ]
        }
    ],
    "resourceProfiles": [
        {
            "type": "kusto",
            "name": "OrbDB",
            "clustersByCloudType": {
                "Public": "https://orbcluster.westus2.kusto.windows.net",
                "Private": "https://orbcluster.westus2.kusto.windows.net"
            },
            "dbsByCloudType": {
                "Public": "orbtestdb"
            },
            "db": "orbtestdb",
            "errorHelpMap": {
                "403": "Please make sure you have valid AAD account."
            }
        }
    ]
}

```

### requiredBaseProps

This is a set of properties that all objects in the namespace will inherit.

>* requiredBaseProps can be used to implement [national clouds support](https://docs.microsoft.com/en-us/graph/deployments)

Currently, only enums are supported as requiredBaseProps. These enum choices are automatically displayed on the search page.

### National Clouds Support

If requiredBaseProps are setup for your namespace as shown in the example above, every object will automatically include the *cloudType* variable.

Kusto/Dgrep resources will be automatically converted to the endpoint defined for the given cloudType.

> Orb will automatically convert saved queries to Public endpoints if the underlying object is in a different cloudType.

# Objects

>* Objects are a collection of resources. These resources are organized as trees called Resource Trees.

## Create a new Object

Under the *Objects* folder for your namespace create a file *objectName*.json.

>* The object json file can be under any subdirectory of *Objects*.
>* An Object can be split into multiple files like *objectName*.file1.json, *objectName*.file2.json. However, *objectName*.json is always required.
>* Protected resources needing Code Reviews (all PowerShell snippets) need to be placed in the ProtectedModels folder. See the Namespace Section above for directory definition examples.

## Global Object json

Global objects do not have any object specific context and do not need any keys for lookups.

Sample global object json file:

```json
{
    "namespace": "OrbSample",
    "path": "Global\\Global Object",
    "resources": [
        {
            "type": "jarvis",
            "relativePath": "Compute\\Global Object",
            "description": "Global Compute VM, Node Availability",
            "link": "https://github.com/Microsoft/Orb"
        }
    ]
}
```

Resource definition details can be found in [this section](#resources)

## Object json

All objects need to specify a path. The path can contain subdirectories.

>* Ideally, the path specified should match the folder structure where the .json file is placed.

Sample object json file:

```json
{
    "namespace": "OrbSample",
    "path": "Compute\\VM",
    "requiredProps": [
        "VMId"
    ],
    "key": "VMId",
    "constructor": {
        "type": "kusto",
        "connectionProfile": "OrbDB",
        "query": "VMSnapshot | where {timeRange} and VMId =~ \"{VMId}\" | take 1 | project VMId",
        "wildcardQuery": "VMSnapshot | where {timeRange} and VMId matches regex \"{VMId}\" | summarize by VMId",
        "minimumResolutionInMinutes": 120
    },
    "additionalProps": [
    ],
    "resources": [
        {
            "type": "kusto",
            "relativePath": "Events\\VM Events (Kusto Example)",
            "connectionProfile": "OrbDB",
            "description": ".kusto example",
            "query": "VMSnapshot | where {timeRange} and VMId == \"{VMId}\" | project PreciseTimeStamp, Event asc"
        },
        {
            "type": "link",
            "relativePath": "VM diagnostics (Link Example)",
            "description": ".link example",
            "link": "https://github.com/Microsoft/Orb?vm={VMId}"
        }
    ],
    "associations": [
        {
            "type": "kusto",
            "relativePath": "Node",
            "associatedObjectPath": "Compute\\Node",
            "description": "",
            "connectionProfile": "OrbDB",
            "query": "NodeSnapshot | where {timeRange} and VMId == \"{VMId}\" | limit 1 | project NodeId",
            "minimumResolutionInMinutes": 120
        }
    ]
}
```

### requiredProps

This is a list of property names that are **required** to construct the object.
>* The requiredProps define the concrete instances of an object.
>* Keep this list to a minimum since every object Search and Association would need to provide all required Props to construct the object.

### key

This is a property from the requiredProps set that is required to search for this object. This is typically the object Id.

### disablePathlessSearch

This prevents the object from being searched when a path is not specified (in the search pane). This defaults to *false* if not defined.
Please use this if your constructor takes a long time to run or requires user input.

### displayName

This overrides the default Orb display name for the specified object. *requiredProps* and *requiredBaseProps* can be subsituted into the display name using the *{requiredProp}* syntax.

### hideFromSearch

This prevents the object from being searched in the search pane. Usefull when you want to make the object available only by associations. The object will be hidden from path and will never be searched. Even if the path is not specified and the `disablePathlessSearch` is set to `false`. The setting defaults to *false* if not defined.

### constructor

>* The constructor is a way to obtain all requiredProps for the object given a key.
>* It is primarily used in the Search view.
>* You can construct Objects using PowerShell or Kusto.

#### Kusto
>* The connection profile should be present in the namespaceConfig.json
>* A wildcard constructor needs to be provided to allow matching on regex inputs
>* Generally, your Kusto queries should end with *project requiredProp1, requireProp2 ...* or *summarize by requiredProp1, requireProp2 ...*

>#### minimumResolutionInMinutes
>
> The *minimumResolutionInMinutes* allows you to specify the minimum time range difference that must be present to run the query.

>* If you are using snapshots to construct objects, set the minimumResolutionInMinutes to at least 2x the snapshot frequency.
>
> For example, if you have an event that emits how many objects exist every 15 minutes, a query with a time range of 5 minutes may not yield any results.
>
> In this case, if you set the min resolution to 30 minutes, for object search to work, the time range in the Search View will automatically expand to be at least 30 minutes, making the search more meaningful.

#### PowerShell

To use Powershell to define objects, see [this](powershellObjects.md) section.

### additionalProps
These are properties that are not required to create the object, but are additional properties on the object.

>* Minimize how many properties are present in requiredProps. More requiredProps makes adding associations to the object much harder.

#### C# Analogy

```csharp

public class MyObject
{
    public MyObject(string requiredProp1, string requiredProp2)
    {
        // This is your constructor. Any one needing to construct this object needs to pass 2 required props.
    }

    public AdditionalProp1
    {
        get
        {
            // This property can be inferred using the required props above.
            return someFunction(requiredProp1, requiredProp2).
        }
    }
}

```

#### constant additionalProps

This allows re-using code/text snippets in the object definition file.

> Constants can only be added under ProtectedModels and must go through Code Reviews. This is because they can be used in any resource, including PowerShell.

```json
{
    "additionalProps": [
        {
            "name": "vmId",
            "type": "constant",
            "value": "{VMId}"
        }
    ]
}
```

#### Kusto additionalProps

Append your object with any number of properties from Kusto.

>* You can pull multiple properties from a single kusto query.

>* AdditionalProps can depend on other required/additional Props, as long as there are no circular dependencies.

```json
{
    "additionalProps": [
        {
            "name": [
                "NodeId"
            ],
            "type": "kusto",
            "connectionProfile": "OrbDB",
            "query": "VMSnapshot | where {timeRange} and VMId == \"{VMId}\" | project NodeId",
            "minimumResolutionInMinutes": 45
        }
    ]
}
```

Now you can use {NodeId} in any resource definition.

#### PowerShell additionalProps

Append your object with a number of properties from PowerShell scripts.

>* You can pull multiple properties from a single PowerShell script.
>* AdditionalProps can depend on other required/additionalProps, as long as there are no circular dependencies.

```json
{
    "additionalProps": [
        {
        "name": [ "ADUser" ],
        "type": "powershell",
        "powershellProfile": "AzurePowerShell",
        "script": "Get-ADUser -Filter * -SearchBase \"OU=Finance,OU=UserAccounts,DC=FABRIKAM,DC=COM\""
        }
    ]
}
```

# Resources

To add a resource to the object, append to the list of resources in object.json.

Every resource can substitute any *requiredProps* in the resource definition.

In addition to *requiredProps*, the following global variables are also available for substitution.

## Global context variables

Variable | Description
------------ | -------------
{startTime} | Start time in UTC that is selected on the Explorer/Search views.
{endTime} | End time in UTC that is selected on the Explorer/Search views.
{timeRange} | Kusto query snippet that can be used in a where clause to combine start/end times. Used in Kusto resources only.

>* Not all resource group types need to use these variables.

>* Some resource handlers automatically inject these variables for you.

## Example of Contextualization

Contextualization is the process of making a generic resource definition a concrete one by replacing *requiredProps* values and global variables.

Sample resource definition:
<pre>
VMSnapshot | where {timeRange} and VMId =~ \"{VMId}\" | take 1 | project VMId
</pre>

Let's say we have a concrete VM with *requiredProp* values.

```json
{
    "requiredProps": {
        "VMId": "df9df236-22fe-4d84-8c6b-42c6d63704e1"
    }
}
```

Also, suppose the time range selected is the last 2 hours.

Output concrete resource:
<pre>
 VMSnapshot | where PreciseTimeStamp > ago(2h) and VMId =~ "df9df236-22fe-4d84-8c6b-42c6d63704e1" | take 1 | project VMId
</pre>

Note all the variable substitutions that occurred above.

## Shared Resource Properties

All resource types share some properties. These are described below.

### type
The type of resource. This must be one of the supported resource types listed below.

### relativePath
The relative path for this resource in explorer. Resources can be organized under arbitrary directories.

### description
A detailed description of the resource.

### showInContextMenu
Setting this to true will show the resource in the right-click context menu instead of explorer.
The relativePath will be converted to sub-menus if it contains any directories.
For example, a relative path of *foo\bar\myResource* will be converted to a menu foo, and sub-menu bar as *foo->bar->myResource*.

## Resource Types

### .link

This is the most basic resource type. It's a general purpose link.

Sample definition:

```json
{
    "type": "link",
    "relativePath": "myDir\\myObject",
    "description": "Sample resource link",
    "link": "https://www.foo.com/?objectId={requiredProp1}"
}
```

### .kusto

This is a kusto query definition.

Use the *{timeRange}* variable in a where clause in the query.

>* Do not hardcode time ranges (relative or absolute) in your query.

Sample definition:

```json
{
    "type": "kusto",
    "relativePath": "Events\\VM Events (Kusto Example)",
    "connectionProfile": "OrbDB",
    "description": ".kusto example",
    "query": "VMSnapshot | where {timeRange} and VMId == \"{VMId}\" | project PreciseTimeStamp, Event asc"
}
```

### .psx

Open any PowerShell script in an external window.

You can specify a PowerShell profile specified in namespaceconfig.json to preload modules for the script.

The profile can be left unspecified if no modules need to be preloaded explicitly.

>* Make sure to escape characters \ (as \\) and " (as \\")
> Must be in ProtectedModels.

```json
{
    "type": "psx",
    "relativePath": "Runtime\\VM State (PSX Example)",
    "description": ".psx example",
    "script": "Write-Host '{vmId} started'"
}
```

### .terminal

Open an Orb powershell terminal and run the provided script. The schema is identical to psx resources.

> Must be in ProtectedModels.

### .psmd

Render any PowerShell script as markdown automatically.

You can specify a PowerShell profile specified in namespaceConfig.json to preload modules for the script.

The profile can be left unspecified if no modules need to be preloaded explicitly.

> Must be in ProtectedModels.

>* Only output, errors and warnings are rendered automatically.

>* Stdout (Write-Host, Write-Verbose) is not displayed.

>* Output format can be set to _auto_, _rawMarkdown_, or _objectExplorer_

>* Object explorer takes a script that returns a PowerShell object and renders the object in an explorable tree.

>* If you specify _objectExplorer_ as an output format, you can also specify _objectExplorerDepth_, which controls how many levels of the tree are initially expanded. If not specified, it expands the tree two levels down.

>* Make sure your scripts return outputs instead of writing to Stdout.

>* Make sure to escape characters \ (as \\) and " (as \\")

```json
{
    "type": "psmd",
    "relativePath": "Runtime\\VM State (PSMD Example)",
    "description": ".psmd example",
    "script": "'{vmId} started'",
    "options": {
        "outputFormat": "auto"
    }
}
```

#### options

The options field is optional.

The outputFormat can be set to *auto*, *objectExplorer* or *rawMarkdown*.

In *auto* the output is formatted automatically. This is the default setting.

If your powershell script output returns custom markdown, use *rawMarkdown*.
This turns off any auto formatting of the output and provides complete control to the PowerShell script to format the output section.
Errors and Warnings returned from the script will still be rendered automatically.

If your script returns a PowerShell object, use *objectExplorer* to enable interactive, tree-based views of the object. You can also
control the default display depth of the tree view using the *formatOptions* tag.

```json
{
    "type": "psmd",
    "relativePath": "ObjectExplorer",
    "powershellProfile": "demops",
    "description": "",
    "script": "$result = $employees|where-object{$_.name -notlike \"{name}*\"}; $result",
    "options": {
        "outputFormat": "objectExplorer",
        "formatOptions": {
            "displayDepth": 0,
            "parseDepth": 4
        }
    }
}
```

If you use the *objectExplorer*, the script you run has *|ConvertTo-JSON* appended to it to pass the data to Orb. To control the depth of the *ConvertTo-JSON* operation, specify the *parseDepth* tag.

# Associations

Object Resource Trees can be linked together using associations.

Kusto queries and PowerShell scripts are supported as associations at this point.

To add an association from *ObjectTypeA* to *ObjectTypeB*:

>* You need to provide a query that can produce all requiredProps of *ObjectTypeB*

In other words, you need to be able to construct objectB given context from objectA.

Sample definition:

```json
{
    "type": "kusto",
    "relativePath": "Events\\VM Events (Kusto Example)",
    "connectionProfile": "OrbDB",
    "description": ".kusto example",
    "query": "VMSnapshot | where {timeRange} and VMId == \"{VMId}\" | project PreciseTimeStamp, Event asc"
}
```

In the above example a Tenant is associated with a Fabric.

For details on minimumResolutionInMinutes see the [constructor section](#constructor) above.

* [Back to Help](all.md)
* [Top](#updating-orb-models)