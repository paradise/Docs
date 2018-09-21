---
uid: config-builder
title: Configuration builders for ASP.NET
author: rick-anderson
description: Learn how to get configuration data from sources other than web.config
ms.author: riande
ms.date: 9/9/2018
ms.technology: aspnet
msc.type: content
---

# Configuration builders for ASP.NET

Configuration builders provide a mechanism for ASP.NET apps to get configuration values. 

Configuration builders:

* Are a new feature of the .Net Framework, introduced in .Net 4.7.1.
* Are flexible.
* Provide a mechanism to get started reading configuration values.
* Address some of the basic needs of applications as they move into a container and cloud focused environment.

## Key/value configuration builders

The most common usage of configuration builders is to provide a basic key/value replacement mechanism. Most of the configuration builders in this project are basic key/value builders. Applications can also use the configuration builder concept to construct complex configuration on the fly.

## Key/value configuration builders settings

The following settings apply to all Key/value configuration builders.

### Mode

The configuration builders use an external source of key/value information to populate selected key/value elements of the configuration system. Specifically, the `appSettings` and `connectionStrings` sections receive special treatment from the configuration builders. These builders can be set to run in three different modes:

* `Strict` - The default mode. In this mode, the configuration builder will only operate on well-known key/value-centric configuration sections. `Strict` mode enumerates each key in the section. If a matching key is found in the external source:

   * The configuration builders replace the value in the resulting configuration section with the value from the external source.
* `Greedy` - This mode is closely related to `Strict` mode. Rather than being limited to keys that already exist in the original configuration:

  * The configuration builders dump all key/value pairs from the external source into the resulting configuration section.

* `Expand` - Operates on the raw XML before it's parsed into a configuration section object. It can be thought of as an expansion of tokens in a string. Any part of the raw XML string that matches the pattern `${token}` is a candidate for token expansion. If no corresponding value is found in the external source, then the token is not changed.

### Prefix handling

Key prefixes can simplify setting keys because:

* The .Net Framework configuration is complex and nested.
* External key/value sources are by nature basic and flat.

For example, use either approach to inject both App Settings and Connection Strings into the configuration via environment variables:

* With the `EnvironmentConfigBuilder` in the default `Strict` mode and use the appropriate key names coded into the configuration file.
* Use two `EnvironmentConfigBuilder`s in `Greedy` mode with distinct prefixes. Using this approach the app can read App Setting and connection strings without needing to update the raw configuration file in advance. For example:

[!code-xml[Main](config-builder/MyConfigBuilders/WebPrefix.config?name=snippet&highlight=11-99)]

With the preceding markup the same flat key/value source can be used to populate configuration for two different sections.

### stripPrefix

`stripPrefix`: boolean, defaults to `false`. 

The preceding code separates app settings from connection strings but requires all the keys to use the prefix. For example, "AppSetting_ServiceID". `stripPrefix` is generally used or remove the prefix added by the `prefix` attribute.

Applications typically strip off the prefix. The following markup strips the prefix:

[!code-xml[Main](config-builder/MyConfigBuilders/WebPrefixStrip.config?name=snippet&highlight=11-99)]

### tokenPattern

`tokenPattern`: string, defaults to `@"\$\{(\w+)\}"`

The `Expand` behavior of the builders search the raw xml for tokens that look like `${token}`. Searching is done with the default regular expression `@"\$\{(\w+)\}"`. The set of characters that matches `\w` is more strict than xml and many sources of configuration values allow. Some apps may need more exotic characters in their token names. There might be scenarios where the `${}` pattern is not acceptable.

`tokenPattern`:

* Allows developers to change the regex that is used for token matching. 
* Is string argument.
* No validation is done to make sure it is a well-formed non-dangerous regex.
* It must contain a capture group. The entire regex must match the entire token. The first capture must be the token name to look up in the configuration source.

## Configuration builders in Microsoft.Configuration.ConfigurationBuilders

### EnvironmentConfigBuilder

```xml
<add name="Environment"
    [mode|prefix|stripPrefix|tokenPattern] 
    type="Microsoft.Configuration.ConfigurationBuilders.EnvironmentConfigBuilder,  Microsoft.Configuration.ConfigurationBuilders.Environment" />
```

The `EnvironmentConfigBuilder`:

* Is the simplest of the configuration builders.
* Draws its values from Environment.
* Does not have any additional configuration options.
* The `name` attribute value is arbitrary.

**NOTE:** In a Windows container environment, variables set at run time are only injected into the EntryPoint process environment. Applications that run as a service or a non-EntryPoint process will not pick up these variables unless they are otherwise injected through some mechanism in the container. For [IIS](https://github.com/Microsoft/iis-docker/pull/41)/[ASP.Net](https://github.com/Microsoft/aspnet-docker)-based
 containers, the current version of [ServiceMonitor.exe](https://github.com/Microsoft/iis-docker/pull/41) handles this in the *DefaultAppPool* only. Other Windows-based container variants may need to develop their own injection mechanism for non-EntryPoint processes.

### UserSecretsConfigBuilder

```xml
<add name="UserSecrets"
    [mode|prefix|stripPrefix|tokenPattern]
    (userSecretsId="{secret string, typically a GUID}" | userSecretsFile="~\secrets.file")
    [optional="true"]
    type="Microsoft.Configuration.ConfigurationBuilders.UserSecretsConfigBuilder,  Microsoft.Configuration.ConfigurationBuilders.UserSecrets" />
```

This configuration builder provides a feature similar to [ASP.NET Core Secret Manager](/aspnet/core/security/app-secrets).

This configuration builder can be used in .NET Framework project, but you must specify a secrets file. Alternatively, you can define the 'UserSecretsId' property in the project file and create the raw secrets file in the correct location for reading. To keep external dependencies out of your project:

* The actual secret file is xml formatted. The xml formatting is an implementation detail, and the format should not be relied upon. 
* If you need to share a *secrets.json* file with .NET Core projects, consider using the [SimpleJsonConfigBuilder(#simplejsonconfig). The `SimpleJsonConfigBuilder` format for .NET Core is an implementation detail subject to change.

Configuration attributes for `UserSecretsConfigBuilder`:

* `userSecretsId` - This is the preferred method for identifying an xml secrets file. It works similar to .Net Core, which uses a 'UserSecretsId' project property to store this identifier. The string must be unique, it does not have to be a GUID. Just unique. With this attribute, the `UserSecretsConfigBuilder` will look in a well-known local location (%APPDATA%\Microsoft\UserSecrets\&lt;userSecretsId&gt;\secrets.xml in Windows environments) for a secrets file belonging to this identifier.
* `userSecretsFile` - An optional attribute specifying the file containing the secrets. The '~' character can be used at the start to reference the application root. Either this attribute or the 'userSecretsId' attribute is required. If both are specified, 'userSecretsFile' takes precedence. <!-- review: isn't ~/ a security risk? .NET Core secret manager stores values in %APPDATA%\Microsoft\UserSecrets so it won't be checked into github -->
* `optional`: boolean, default value `true` - Prevents an exception if the secrets file cannot be found. 
* The `name` attribute value is arbitrary.

The secrets file has the following format:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<root>
  <secrets ver="1.0">
    <secret name="secret key name" value="secret value" />
  </secrets>
</root>
```

### AzureKeyVaultConfigBuilder

```xml
<add name="AzureKeyVault"
    [mode|prefix|stripPrefix|tokenPattern]
    (vaultName="MyVaultName" |
     uri="https://MyVaultName.vault.azure.net")
    [connectionString="connection string"]
    [version="secrets version"]
    [preloadSecretNames="true"]
    type="Microsoft.Configuration.ConfigurationBuilders.AzureKeyVaultConfigBuilder,  Microsoft.Configuration.ConfigurationBuilders.Azure" />
```

`AzureKeyVaultConfigBuilder` reads values stored in the [Azure Key Vault](/azure/key-vault/key-vault-whatis).

If your secrets are kept in Azure Key Vault, then this configuration builder is for you. There are three additional attributes for this configuration builder. The `vaultName` is required. The other attributes allow you some manual control about which vault to connect to, but are only necessary if the application is not running in an environment that works magically with `Microsoft.Azure.Services.AppAuthentication`. The Azure Services Authentication library is used to automatically pick up connection information from the execution environment if possible, but you can override that feature by providing a connection string instead.
* `vaultName` - This is a required attribute. It specifies the name of the vault in your Azure subscription from which to read key/value pairs.
* `connectionString` - A connection string usable by [AzureServiceTokenProvider](https://docs.microsoft.com/en-us/azure/key-vault/service-to-service-authentication#connection-string-support)
* `uri` - Connect to other Key Vault providers with this attribute. If not specified, Azure is the assumed Vault provider. If the uri _is_specified, then `vaultName` is no longer a required parameter.
* `version` - Azure Key Vault provides a versioning feature for secrets. If this is specified, the builder will only retrieve secrets matching this version.
* `preloadSecretNames` - By default, this builder will query **all** the key names in the key vault when it is initialized. If this is a concern, set
this attribute to 'false', and secrets will be retrieved one at a time. This could also be useful if the vault allows "Get" access but not
"List" access. (NOTE: Disabling preload is incompatible with Greedy mode.)

### KeyPerFileConfigBuilder

```xml
<add name="KeyPerFile"
    [mode|prefix|stripPrefix|tokenPattern]
	(directoryPath="PathToSourceDirectory")
    [ignorePrefix="ignore."]
    [keyDelimiter=":"]
    [optional="false"]
    type="Microsoft.Configuration.ConfigurationBuilders.KeyPerFileConfigBuilder,  Microsoft.Configuration.ConfigurationBuilders.KeyPerFile" />
```
This is a simple configuration builder that uses a directory's files as a source of values. A file's name is the key, and the contents are the value. This
configuration builder can be useful when running in an orchestrated container environment, as systems like Docker Swarm and Kubernetes provide 'secrets' to
their orchestrated windows containers in this key-per-file manner.
* `directoryPath` - This is a required attribute. It specifies a path to the source directory to look in for values. Docker for Windows secrets
are stored in the 'C:\ProgramData\Docker\secrets' directory by default.
* `ignorePrefix` - Files that start with this prefix will be excluded. Defaults to "ignore.".
* `keyDelimiter` - If specified, the configuration builder will traverse multiple levels of the directory, building key names up with this delimiter. If
this value is left `null` however, the configuration builder only looks at the top level of the directory. `null` is the default.
* `optional` - Specifies whether the configuration builder should cause errors if the source directory doesn't exist. The default is `false`.

### SimpleJsonConfigBuilder

```xml
<add name="SimpleJson"
    [mode|prefix|stripPrefix|tokenPattern]
    jsonFile="~\config.json"
    [optional="true"]
    [jsonMode="(Flat|Sectional)"]
    type="Microsoft.Configuration.ConfigurationBuilders.SimpleJsonConfigBuilder,  Microsoft.Configuration.ConfigurationBuilders.Json" />
```

Because .Net Core projects can rely heavily on json files for configuration, it makes some sense to allow those same files to be used in full-framework
configuration as well. You can imagine that the hierarchical nature of json might enable some fantastic capabilities for building complex configuration sections.
But this configuration builder is meant to be a simple mapping from a flat key/value source into specific key/value areas of full-framework configuration. Thus its name
begins with 'Simple.' Think of the backing json file as a simple dictionary, rather than a complex hierarchical object.

(A multi-level hierarchical file can be used. This provider will simply 'flatten' the depth by appending the property name at each level using ':' as a delimiter.)

There are three additional attributes that can be used to configure this builder:
* `jsonFile` - A required attribute specifying the json file to draw from. The '~' character can be used at the start to reference the app root.
* `optional` - A simple boolean to avoid throwing exceptions if the json file cannot be found. The default is `true`.
* `jsonMode` - `[Flat|Sectional]`. 'Flat' is the default.

    - This attribute requires a little more explanation. It says above to think of the json file as a single flat key/value source. This is the usual that applies to other key/value configuration builders like `EnvironmentConfigBuilder` and `AzureKeyVaultConfigBuilder` because those sources provide no other option. If the `SimpleJsonConfigBuilder` is configured in 'Sectional' mode, then the json file is conceptually divided just at the top level into multiple simple dictionaries. Each one of those dictionaries will only be applied to the configuration section that matches the top-level property name attached to them. For example:
```json
    {
        "appSettings" : {
            "setting1" : "value1",
            "setting2" : "value2",
            "complex" : {
                "setting1" : "complex:value1",
                "setting2" : "complex:value2",
            }
        },

        "connectionStrings" : {
            "mySpecialConnectionString" : "Dont_check_connection_information_into_source_control"
        }
    }
```

## Implementing More Key/value Config builders

If you don't see a configuration builder here that suits your needs, you can write your own. Referencing the `Basic` NuGet package for this project will get you the base upon which
all of these builders inherit. Most of the heavy-ish lifting and consistent behavior across key/value configuration builders comes from this base. Take a look at the code for more
detail, but in many cases implementing a custom key/value configuration builder in this same vein is as simple as inheriting the base, and implementing two simple methods.
```CSharp
using Microsoft.Configuration.ConfigurationBuilders;

public class CustomConfigBuilder : KeyValueConfigBuilder
{
    public override string GetValue(string key)
    {
        // Key lookup should be case-insensitive, because most key/value collections in .Net config sections are as well.
        return "Value for given key, or null.";
    }

    public override ICollection<KeyValuePair<string, string>> GetAllValues(string prefix)
    {
        // Populate the return collection a little more smartly. ;)
        return new Dictionary<string, string>() { { "one", "1" }, { "two", "2" } };
    }
}
```

## How to contribute

## Blog Posts
[Announcing .NET 4.7.1 Tools for the Cloud](https://blogs.msdn.microsoft.com/webdev/2017/11/17/announcing-net-4-7-1-tools-for-the-cloud/)  
[.Net Framework 4.7.1 ASP.NET and Configuration features](https://blogs.msdn.microsoft.com/dotnet/2017/09/13/net-framework-4-7-1-asp-net-and-configuration-features/)  
[Modern Configuration for ASP.NET 4.7.1 with ConfigurationBuilders](http://jeffreyfritz.com/2017/11/modern-configuration-for-asp-net-4-7-1-with-configurationbuilders/)  
[Service-to-service authentication to Azure Key Vault using .NET](https://docs.microsoft.com/en-us/azure/key-vault/service-to-service-authentication#connection-string-support)  