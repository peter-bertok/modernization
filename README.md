# Checklist for Light-Touch App Modernization
The following is relevant to a typical legacy ASP.NET web application being 
migrated to the public cloud. It's most applicable when moving to a true 
Platform-as-a-Service (PaaS) offering such as Azure App Service, but is equally
applicable to container-platforms such as Container Apps or Kubernetes. Most of
the steps are also highly recommended in general, even if moving to virtual machines.
About 80% of the steps are required if moving to a VM Scale Set via DevOps pipeline
deployments.

## General Fixup
This ensures compatibility and supportability with cloud services and platforms. For example,
most cloud SDKs are written using .NET Standard 2.0, which requires an update to .NET 4.7.2 at
a minimum. Full support for containers requires .NET 4.8, which includes some fixes for critical
issues such as excessive memory usage in older versions.

- [ ] Ensure all projects are compatible with Visual Studio 2022.
- [ ] Upgrade 4.x projects to .NET 4.8. This is configured in 3x places:
  - [ ] web.config compilation settings
  - [ ] web.config httpRuntime settings
  - [ ] packages.config
- [ ] Update NuGet package versions where possible and/or safe.
  - [ ] Focus effort on packages with security warnings.
  - [ ] Anything that's a pre-requisite for cloud integration such as "Azure.Identity".
- [ ] Optionally update JQuery (where possible)
  - [ ] Older versions used are designed for IE6, have various issues in modern browsers.
  - [ ] This is best implemented after the site is *testable*, use F12 tools to validate!
- [ ] If needed, disable "Unobtrusive JavaScript" control validation.
- [ ] Disable any hard-coded uses of corporate HTTP web proxies.
- [ ] Fix ASP.NET cookie security flags: secure, HTTP-only, correct DNS domain.
  - [ ] In the global web.config settings
  - [ ] Also in the "Forms Authentication" settings.
- [ ] Update all back-end server names to use FQDNs (e.g.: "prdappserver1" -> "prdappserver1.internal.corp")
- [ ] Update all URL references to use HTTPS where possible
  - [ ] Headers, footers, redirect targets, WCF/ASMX references, JavaScript CDN URLs, etc...
  - [ ] Ideally site-relative, e.g.: ("~/foo/script.js")
- [ ] Remove any hard-coded system paths ("D:\logs", "C:\temp", etc...)
  - [ ] Replace with app-relative paths and/or GetTempDirectory() and the like         
- [ ] Remove stale/unused config settings.
- [ ] Update/remove any references to external EXEs launched by the web apps  
  - [ ] No "C:\Program Files..." paths should be used. Must be app-relative!
  - [ ] Include the EXE in the Git repo (if small)
  - [ ] Or: include in the build pipeline Zip artefact (if large).
- [ ] Assume that the platform will be UTC and en-US.
  - [ ] Hard-code globalization settings in web.config.
  - [ ] Ensure App Service time zone is set.
  - [ ] Use Windows Server 2022 container images, which support per-container time zones
- [ ] Ensure you are truly building your application! ASP.NET Web Forms apps are *compiled on the IIS server* by default. Old apps often have broken ASPX pages that throw HTTP/500 "internal server errors" to *end users* weeks later because of platform compatibility issues.

Example lines from a fixed web.config:

```xml
    <compilation debug="true" targetFramework="4.8" />
    <httpRuntime targetFramework="4.8"/>    
    <globalization requestEncoding="utf-8" responseEncoding="utf-8" culture="en-AU" uiCulture="en-AU"/>
    <httpCookies httpOnlyCookies="true" requireSSL="true"/>

    <!-- if needed -->
    <pages controlRenderingCompatibilityVersion="4.5" clientIDMode="AutoID"/>  

    <!-- if needed -->
    <appSettings>  
      <add key="ValidationSettings:UnobtrusiveValidationMode" value="None"></add>  
    </appSettings>  
```

## Parameterisation and Hygienic Deployments
Many older web apps were "edited in-place", including direct edits to web.config files, manual
uploads of static content, and incremental deployments where individual code files such as ASPX
pages were uploaded to servers. Worse still, some web apps contain user data blended in with app
code.

An aspirational goal is for the deployed web app to act more like a Docker container image: an immutable
binary blob that can be moved from environment-to-environment with a simple file copy, where a deployment
*entirely replaces* the existing one in a single, atomic step. This doesn't actually require a container
image and can be implemented on traditional virtual machines with the IIS web server platform. The concept
is that it should be possible for a developer to delete the contents of a virtual folder at *any time*, and
then simply copy in the content from a *different environment* and then that's... it. No further steps
should be required. 

- [ ] Move all environment-specific configuration settings into the `<AppSettings>` or `<ConnectionStrings>` sections, which IIS, App Service, and Docker can control from "the outside". Notably, the `<ApplicationSettings>` setting used by WCF is not easy to configure!
- [ ] If any files such as JavaScript varies per-environment, generate them dynamically in ASPX if possible.
- [ ] Avoid hard-coded redirects in web.config. These are served by IIS, not ASP.NET, and cannot be configured via AppSettings.
- [ ] For ASP.NET Core apps, use the [ASPNETCORE_ENVIRONMENT](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/environments?view=aspnetcore-8.0#environments) flag to switch configs dynamically.
- [ ] Consider using Azure App Configuration, or a similar "external" configuration source.

## DevOps 101
Many older applications are built manually using Visual Studio, depend on components
installed "system-wide", or require manual steps to package, deploy, and configure.

While some PaaS platforms do support manual or semi-manual workflows, they're designed
for modern "dev-ops" workflows such as CI/CD, cloud-hosted automated builds, automated
integration tests, automated deployments, and active health monitor probes in production.

Fixing this is not 100% required, but it is *highly recommended* because old apps
that may have remained unmodified for years will often be updated several times
immediately after a migration. For example, updating logos, styling, content, or 
integrations with services such as Azure AD B2C. Making the deployment process smooth
reduces the cost of these changes.

- [ ] Ensure all files required for builds and deployments are in the source control management system (SCM).
- [ ] Ideally, use Azure DevOps or GitHub when deploying to Azure.
- [ ] Migrate to Git SCM if not already using it. Many older projects use Team Foundation Version Control (TFVC), which modern tools such as NBGV don't support.
- [ ] Builds must be successful on a "blank", isolated machine with only Visual Studio 2022 on it. (I.e.: an Azure DevOps pipeline agent.)
- [ ] Deployments must not require any manual steps.
- [ ] The "web.config" in the source code should match the one in production (in structure, but not specific settings).
- [ ] Create build pipelines. Make sure to use the new "2.0" pipelines.
  - [ ] If you can't see a YAML file in the source code, you went wrong somewhere.
  - [ ] If you use separate "build" and "release" pipelines, this is the old 1.0 style GUI-wizard pipelines.
  - [ ] Use "pipeline artifacts" for zip files.
  - [ ] Use separate "stages" for build and each deployment. E.g.: Build -> TST -> UAT -> PRD.
  - [ ] Use "Environments" with security rules associated with the PRD environment to block accidental deployments to the live environment.
  - [ ] Use branch security and similar controls to ensure that only the "main" or "master" branch deploys to PRD.
- [ ] Find NuGet packages for DLL binary files and replace the DLL file with the package reference.
  - [ ] "Vendor" the dependency into Azure DevOps. This is a checkbox feature called "Package Cache".
  - WARNING: There are no official packages for Crystal Reports!
- [ ] Internally-developed DLLs from other projects should be published as NuGet packages to a private feed hosted in Azure DevOps.
- [ ] Web apps should be [precompiled during builds](https://learn.microsoft.com/en-us/aspnet/web-forms/overview/older-versions-getting-started/deploying-web-site-projects/precompiling-your-website-cs), otherwise some web pages might not actually function on PaaS, but this won't be noticed until production when they're "compiled on the fly".
- [ ] Add the "Source Indexing" task to the pipeline, which then enables Application Insights debug snapshots to automatically download the matching code files.
- [ ] Publish PDB files even in release builds to the PaaS platform. Check the "zip" artefact files to make sure they're there. These are used by both App Service and Application Insights for diagostics such as crash dump analysis and memory leak analysis.

In Azure DevOps Pipelines, this means adding the following snippets to the msbuild args:

```batch
    /p:DebugType=Full /p:PrecompileBeforePublish=true /p:EnableUpdateable=false /p:DebugSymbols=true
```

Also add the following task immediately after the build step to enable source indexing:

```YAML
    - task: PublishSymbols@2
      inputs:
        SymbolsFolder: '$(Build.ArtifactStagingDirectory)'
        SearchPattern: '**/bin/**/*.pdb'
        SymbolServerType: 'TeamServices'
        TreatNotIndexedAsWarning: true
```

## Cloud Integrations
The following changes are technically optional, but very highly recommended because they improve both migrations and ongoing operations.

- [ ] Add a "health monitor" endpoint such as "/healthz". This is used by App Service and other load balancers to identify healthy instances.
  - [ ] For ASP.NET Core apps [there is a standard methodology](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks?view=aspnetcore-8.0).
  - [ ] For .NET Framework 4.x apps a good approach is to add a new [IHttpHandler](https://learn.microsoft.com/en-us/dotnet/api/system.web.ihttphandler?view=netframework-4.8.1) bound to the same path.
  - [ ] Ideally, filter this path such that it only responds on the "local network" and does not respond to requests from the Internet.
- [ ] Where possible, configure service clients to use Azure managed identities for authentication. This eliminates the use of passwords or the requirement for key rotation. This will likely require the use of an updated client library, such as the latest [Microsoft.Data.SqlClient](https://learn.microsoft.com/en-us/sql/connect/ado-net/introduction-microsoft-data-sqlclient-namespace).
- [ ] Alternatively, move secrets into Azure Key Vault or DevOps "secret" parameters. For local development, use "User Secrets" files. There is a right-click wizard for this setup in Visual Studio.
- [ ] Avoid catching and ignoring exceptions. E.g.: empty ```catch {}``` blocks are generally a problem and should be removed in almost all cases.
  - Not allowing exceptions to bubble up to the ASP.NET request pipeline will *falsely* report HTTP 200/OK to upstream systems such as App Service, Application Insights, and reverse proxies such as Azure Front Door.
  - Error counts, alerts, crash dumps, and other diagnostics from Application Insights will be missing and non-functional.
  - Client are then likely to inadvertently cache failure error messages, leading to persistent errors that can only be resolved by customers "clearing their browser cache".
  - Applications that report HTTP 200/OK for errors are generally incompatible with CDNs or any kind of caching. Developers are forced to disable all caching, harming performance.
- [ ] Ensure that all key parameters are configurable from the "outside world", e.g. via environment varibles or App Service configuration settings. The latter is required for [using deployment slots](https://learn.microsoft.com/en-us/azure/app-service/deploy-staging-slots).
