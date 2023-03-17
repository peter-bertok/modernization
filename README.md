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
- [ ] If needed, disable "Unobtrustive JavaScript" control validation.
- [ ] Disable any hard-coded uses of corporate HTTP web proxies.
- [ ] Fix ASP.NET cookie security flags: secure, HTTP-only, correct DNS domain.
  - [ ] In the global web.config settings
  - [ ] Also in the "Forms Authentication" settings.
- [ ] Update all back-end server names to use FQDNs (e.g.: "prdappserver1" -> "prdappserver1.internal.corp")
- [ ] Update all URL references to use HTTPS where possible
  - [ ] Headers, footers, redirect targets, WCF/ASMX references, JavaScript CDN URLs, etc...
  - [ ] Ideally site-relative, e.g.: ("~/foo/script.js")
- [ ] Remove any hard-coded system paths (D:\logs", "C:\temp", etc...)
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


## DevOps 101
Many older applications are built manually using Visual Studio, depend on components
installed "system-wide", and require manual steps to package, deploy, and configure.

While some PaaS platforms do support manual or semi-manual workflows, they're designed
for modern "dev-ops" workflows such as CI/CD, cloud-hosted automated builds, tests, and
deployments. 

Fixing this is not 100% required, but is *highly recommended* because old apps
that may have remained unmodified for years will often be updated several times
immediately after a migration. For example, updating logos, styling, content, or 
integrations with services such as Azure AD B2C.

- [ ] Ensure all files required for builds and deployments are in a source control management system (SCM).
- [ ] Ideally, use Azure DevOps or GitHub when deploying to Azure.
- [ ] Migrate to Git SCM if not already using it. Many older projects use Team Foundation Version Control (TFVC), which modern tools such as NBGV don't support.
- [ ] Builds must be successful on a "blank", isolated machine with only Visual Studio 2022 on it. (I.e.: an Azure DevOps pipeline agent.)
- [ ] Deployments must not require any manual steps.
- [ ] The "web.config" in the source code should match the one in production (in structure, but not specific settings).
- [ ] Create build pipelines. Make sure to use the new "2.0" pipelines.
  - [ ] If you can't see a YAML file in the source code, you went wrong somewhere.
  - [ ] If you use seperate "build" and "release" pipelines, this is the old 1.0 style GUI-wizard pipelines.
  - [ ] Use "Environments" with security rules associated with the PRD environment.
  - [ ] Use "pipeline artefacts" for zip files.
  - [ ] Use seperate "stages" for build and each deployment. E.g.: Build -> TST -> UAT -> PRD.
- [ ] Find NuGet packages for DLL binary files and replace the DLL file with the package reference.
  - WARNING: There are no official packages for Crystal Reports!
- [ ] Internally-developed DLLs from other projects should be published as NuGet packages to a private feed hosted in Azure DevOps.
 

