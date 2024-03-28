# Umbraco Container Example
This repository is an example of how you may create your own Umbraco instance using Webslice Containers. We recommend creating your own repository, however this can serve as an example of how you may want to create your Umbraco project.

This document will assume your local system is using Ubuntu Linux (22.04 LTS Jammy). Your instructions may be different depending on your system. On the container side, the instructions should remain the same.

## Requirements
First you'll need to install the .NET SDK onto your local system.
For Ubuntu linux you can install it via snap using `snap install --classic dotnet-sdk` Depending on your version, distro and OS you will need to run your own commands.
After that, you will need to install Umbraco `dotnet new -i Umbraco.Templates`.
You can choose which version you wish to use like so. `dotnet new -i Umbraco.Templates:10.3.2`
> Note we are currently only able to support Umbraco version 10 and up. Umbraco version below 10 requires .NET versions lower than 6 which is not available as a standard container image.

For our Webslice Container, we will want to create a SSH user that GitHub will use to connect to our container. We recommend creating a new user specifically for GitHub in the event of a compromise.

## Creating the Umbraco Project
To create the project you will want to run the following command: `dotnet new umbraco --name <Project Name/Directory>`
Once that is done you will now have a new Umbraco project under the same name as the project name.
There's a couple of changes we need to make in the file `Properties/launchSettings.json`.
Under `issSetings`, you will need to change `applicationUrl` to `http://localhost`. After that you will need to change `sslPort` to `80`.
Next under `Umbraco.Web.UI` you will need to remove all of the URLs in `applicaationUrl` and leave the list empty.

After all of that, your `launchSettings.json` should look like this.
```json
{
  "$schema": "https://json.schemastore.org/launchsettings.json",
  "iisSettings": {
    "windowsAuthentication": false,
    "anonymousAuthentication": true,
    "iisExpress": {
      "applicationUrl": "http://localhost"
    }
  },
  "profiles": {
    "IIS Express": {
      "commandName": "IISExpress",
      "launchBrowser": true,
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "Umbraco.Web.UI": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "applicationUrl": "",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  }
}
```

Lastly for our project, we'll need to ensure that Umbraco doesn't run the installation wizard. You can do this step after you have deployed the project first onto a Webslice Container or before hand where you'll have to define some variables.

We'll start with the latter. To start, we'll need our default admin login details. You'll need to add in the following underneath `CMS`.
```json
  "Umbraco": {
    "CMS": {
      "Unattended": {
        "InstallUnattended": true,
        "UnattendedUserName": "tester",
        "UnattendedUserEmail": "tester@test.te",
        "UnattendedUserPassword": "P@ssw0rd"
      },
```
Next you'll need to enter a `ConnectionString`. For this example, we'll assume you're using SQLite.
```json
  "ConnectionStrings": {
    "umbracoDbDSN": "Data Source=|DataDirectory|/Umbraco.sqlite.db;Cache=Shared;Foreign Keys=True;Pooling=True",
    "umbracoDbDSN_ProviderName": "Microsoft.Data.Sqlite"
  }
```
For more information on ConnectionStrings, you can check out Umbraco's documentation [here](https://docs.umbraco.com/umbraco-cms/reference/configuration/connectionstringssettings).
With those two done. Your `appsettings.json` file should look something like this:
```json
{
  "$schema": "appsettings-schema.json",
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "Microsoft.Hosting.Lifetime": "Information",
        "System": "Warning"
      }
    }
  },
  "Umbraco": {
    "CMS": {
      "Unattended": {
        "InstallUnattended": true,
        "UnattendedUserName": "tester",
        "UnattendedUserEmail": "tester@test.te",
        "UnattendedUserPassword": "P@ssw0rd"
      },

      "Global": {
        "Id": "9580700d-ac3c-4039-876a-d79fa92bf392",
        "SanitizeTinyMce": true
      },
      "Content": {
        "AllowEditInvariantFromNonDefault": true,
        "ContentVersionCleanupPolicy": {
          "EnableCleanup": true
        }
      }
    }
  },
  "ConnectionStrings": {
    "umbracoDbDSN": "Data Source=|DataDirectory|/Umbraco.sqlite.db;Cache=Shared;Foreign Keys=True;Pooling=True",
    "umbracoDbDSN_ProviderName": "Microsoft.Data.Sqlite"
  }
}
```

If you want to run the Installation wizard, run the wizard once, take the `appsettings.json` file from the deployed website directory then add `Unattended` object with only`InstallUnattended` inside. You do not need to add the admin user details.

## Setting up the Container
For our Webslice Container, we need to make some changes. The first change we need to make is to clear our the `/container/application/public` directory. A simple `rm -rf /container/application/public/*` will do the trick.
Next we'll need to change our supervisor config under `/container/config/supervisord.conf`.
The only line you need to edit is to change `command=/usr/bin/dotnet Example.dll` to `command=/usr/bin/dotnet <Project Name>.dll`.

## Configuring up GitHub Actions
After creating a GitHub repo, what you can do now is create your workflow file. We recommend adjusting your workflow to your project's requirements.
For the sake of this example, we will provide a workflow file that you can find under `.github/workflows/umbraco.yml-example`. Ensure to rename to file by removing the `-example` once you have made your changes.
The workflow file here has a couple of environment variables we will need to add into our GitHub repository.

Go into your repository's `Settings` tab then go into `Secrets and variables` then into the subcategory `Actions`.

Once there you will need to add three environment variables. `SSH_HOST`, `SSH_USER`, `SSH_KEY`. `SSH_PASSWORD` is also available if you want to use a password instead of a private/public key (although you will need to uncomment a line from the workflow file).

You will want to add the IP address for your Webslice Container into `SSH_HOST`, your SSH username into `SSH_USER` and your choice between private key into `SSH_KEY` or password into `SSH_PASSWORD`.

## Finishing up
To make sure the initial set up of the wizard is gone through, you will want to download the `appsettings.json` from the server and copy it over to the repository. This will save the configurations you've made through the wizard as without saving it, every time you compile the project, it will overwrite those settings with the default settings.

After this, any push or pull request that is done to the repository the Action will run. This will build, test and then upload the compiled build up to the container. You will need to restart the Container from the Webslice Console for the changes to take effect.

## Adding new Packages
To add new packages, you can simply add them like other .NET packages. If you go to the Packages tab underneath your Umbraco control panel, you can click on whichever package you want and it will give you the command to run. Do this in your repository, commit and push the changes then GitHub actions will push the Package to your server.

## Enforcing HTTPS
You can check out our [.NET/ASP.NET SSL documentation](https://webslice.com/docs/containers/features/ssl/#netaspnet) for more information.
