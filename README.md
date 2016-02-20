Automatic Chocolatey Package Update Module
==========================================

This Powershell module implements functions that can be used to automate [Chocolatey](https://chocolatey.org) package updates.

It can be used instead of the [official method](https://github.com/chocolatey/choco/wiki/AutomaticPackages).

Installation
------------

Copy Powershell module to any of the directories in the $Env:PSModulePath.

Creating the package updater script
-----------------------------------

- In the package directory, create the script `update.ps1`.
- Import the module: `import-module au`
- Implement two global functions:
  - `au_GetLatest`   
  Function returns HashTable with the latest remote version along with other arbitrary user data which you can use elsewhere (for instance in search and replace). The returned version is then compared to the one in the nuspec file and if they are different, the files will be updated. This hashtable is available via global variable `$Latest`.
  - `au_SearchReplace`  
  Function returns HashTable containing search and replace data for any file in the form: 
  ~~~
    @{ 
        file_path1 = @{ 
            search1 = replace1
            ...
            searchN = replaceN 
        }
        file_path2 = @{ ... }
    }
  ~~~

- Call the `update` function from the `au` module to update the package.

This is best understood via the [example](https://github.com/majkinetor/chocolatey/blob/master/dngrep/update.ps1).

With this set, you can call individual `update.ps1` from within its directory to update that specific package.

Updating all packages
---------------------

You can update all packages and optionally push them to the chocolatey repository with a single command. 

Function `Update-AUPackages` will iterate over `update.ps1` scripts and execute each. If it detects that package is updated it will `cpack` it and push it. 
The function does some rudimentary verifications of URLs and version strings.
For push to work, specify your API key in the file `api_key` in the script's directory.

This function is designed for scheduling. You can use `Install-AUScheduledTask` to install daily scheduled task that points to the `update_all.ps1` script. In this scenario, we want to be notified about possible errors during packages update procedure. If the update procedure fails for any reasons there is an option to send an email with results as an attachment in order to investigate the problem. This is the prototype of the `update_all.ps1`:

    param($Name = $null)
    cd $PSScriptRoot
    import-module au

    $options = @{
        Mail = @{
            To       = 'meh@gmail.com'
            Server   = 'smtp.gmail.com'
            UserName = 'meh@gmail.com'
            Password = '**************'
            Port     = 587
            EnableSsl= $true
        }
    }

    Update-AUPackages -Name $Name -Options $options | Export-CliXML update_results.xml

Use function parameter `Name` to specify package names via glob, for instance "d*" would update only packages which names start with the letter 'd'. Add `Push` among options to push sucesifully built packages to the chocolatey repository. The result may look like this:

    PS C:\chocolatey> .\update_all.ps1

    Updating all automatic packages
    copyq is updated to 2.6.1 and pushed 
    dngrep had errors during update
        Can't validate URL 'https://github.com/dnGrep/dnGrep/releases/download/v2.8.16.0/dnGREP.2.8.16.x64.msi'
        Exception calling "GetResponse" with "0" argument(s): "The operation has timed out"
    eac has no updates
    pandoc has no updates
    plantuml has no updates
    yed had errors during update
        Can't validate URL 'https://www.yworks.com'
        Invalid content type: text/html

    Automatic packages processed: 6
    Total errors: 2
    Mail with errors sent to meh@gmail.com

The email attachment is a `$result` object that keeps all the information about each package which happened during update. It can be loaded with `Import-CliXml` and inspected.

Other functions
---------------

- Test-Package  
Quickly cpack and install the package from the current directory

- Push-Package  
Push the latest package using API key in the api_key file

- Get-AuPackages
Returns the list of the packages which have `update.ps1` script in its directory.
