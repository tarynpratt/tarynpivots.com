---
title: Deploying My SQL Server Maintenance Scripts
date: 2020-10-16 04:00:00 +0000
draft: false 
tags: [sql, sql-server, stack-overflow, performance-tuning, monitoring, maintenance, powershell]
excerpt: How I deploy my SQL Server Toolbox
---

In my [previous post](https://www.tarynpivots.com/post/2020/my-sql-server-toolbox/), I listed out the tools I use with SQL Server. Some of the tools are SQL scripts that need to be deployed to each server. If you have 1-2 SQL Servers, manually deploying scripts might not be bad, but ideally you don't want to manually deploy anything, so I wrote a little script that allow me to install the SQL scripts in my toolbox to any environment. 

## Background

When I started as the DBA at Stack Overflow, our SQL Servers were kind of a mess (sorry to those who maintained them before me, but they were). Many of the servers hadn't been patched in a long time, and we had different versions of various SQL scripts like, [sp_whoisactive](http://whoisactive.com/) and Brent Ozar's [First Reponder Kit](https://www.brentozar.com/responder/),  everywhere. Since the SQL Servers were now my responsibility I wanted everything in order. I decided that one of my first tasks would be to create a simple process to deploy the same scripts everwhere. 

## My Little Deploying Scripts Project

When I started the project, I had already decided what maintenance tools I wanted to install across our environments - these are most of the SQL scripts in my toolbox - I just needed to figure out how I was going to get them on our 30 servers at the same time. 

After several years away from SQL Server and PowerShell, I needed to get my feet wet again, so I decided to write a very basic process that would do the following:

1. Put the .sql files somewhere accessible 
2. Get the list of servers to install on
3. Loop through each server and deploy all the .sql files

### Storing the SQL Scripts

I decided to put the .sql files in a centralized location that all of the servers would be able to access. This was easy enough because we already had infrastructure in place that could be used for this purpose. All I needed to do was pick a spot and throw all the files in a directory. 

Once the files were in the location, I would just loop through them and execute each one. Step 1 done. 

### Gathering the List of SQL Servers

Step 2 in the project was to get a list of all the SQL Servers I wanted the tools on. Easy enough, right? But then I needed a way to parse that list in PowerShell. I wasn't exact sure how I wanted to do this. 

I looked at some of the other tools being used at Stack Overflow for ideas. The SRE team uses [puppet](https://puppet.com/) to deploy to our infrastructure, and puppet uses JSON files to store key-value pairs. I wondered if I could do the same for this tiny little project. 

After several failed attempts at getting the JSON right, I was able to use the following format to store the info about the SQL Servers I wanted to deploy to:

{{< highlight json>}}
{
    "ScriptDirectory":"\\\\put\\your\\directory\\name\\here\\",
    "servers":[
        {
            "name":"sqlserver01"
            ,"type":"zone1_servers"
        },
        {
            "name":"sqlserver02"
            ,"type":"zone3_servers"
        },
        {
            "name":"sqlserver03"
            ,"type":"zone4_servers"
        },
        {
            "name":"protectedsqlserver01"
            ,"type":"zone2_servers"
        }
    ]
}
{{< / highlight >}}

In the config.json file, I'm storing the `ScriptDirectory` which is the path I put all of the .sql files to install. Next, it contains an array of the list of servers. Within the array, I'm storing the `name` of the server and then the `type` of the server. The `name` is pretty self-explanatory, just include each server name and you're done. 

The `type` might seem slightly odd since these are all SQL Servers, but there was a reason I needed this. We have some servers protected by different firewalls, for example the servers that hold the Stack Overflow and the Stack Exchange network of databases are in a totally different zone, than the servers for [Stack Overflow for Teams](https://stackoverflow.com/teams). My PowerShell script would need to be able to distinguish between the multiple firewalls, so I used this as the identifier.

Now that I had a way to get all the servers to push to Step 2 was done. 

### My Little PowerShell Script

The last part of the project was to put all the pieces together and write the PowerShell script to deploy everything automatically. I wrote the script to accept a couple of parameters:

- `$DeploySecure` - this parameter is required and is used to direct the deployment to the specific firewalled areas I mentioned above
- `$ServerGroup` - this is an optional parameter and will let you deploy to specific sets of servers based on their `type` in the config

Keep in mind this script is about 3 years old, and I know it could be written much better than it is.

{{< highlight powershell>}}
<#
.SYNOPSIS
    Script for deploying SQL Server maintenance scripts across environments

.EXAMPLE
    # for zone 1 environments
    .\Deploy-MaintenanceScripts.ps1 -DeploySecure $false

    # for zone 2 environments
    .\Deploy-MaintenanceScripts.ps1 -DeploySecure $true
#>

[CmdletBinding()] #See http://technet.microsoft.com/en-us/library/hh847884(v=wps.620).aspx for CmdletBinding common parameters
param(
    [parameter(Mandatory = $true)]
    [bool]$DeploySecure = $false,
    [parameter(Mandatory = $false)]
    [string]$ServerGroup
)

$Config = Get-Content .\Config.json | ConvertFrom-Json 

if ($DeploySecure -eq $false) {
    $Servers = $Config.servers | Where-Object {$_.type -ne "zone2_servers"}
    Write-Host "Deploying to non-zone 2 environments."
}
else {
    $Servers = $Config.servers | Where-Object {$_.type -eq "zone2_servers"}
    Write-Host "Deploying to zone 2 environments."
}

if ($ServerGroup){
    $Servers = $Config.servers | Where-Object {$_.type -eq $ServerGroup}
    Write-Host "Deploying servers to $ServerGroup"
}

$ScriptDirectory = $Config.ScriptDirectory

If(-not (Test-Path $ScriptDirectory)){
    Write-Error "$ScriptDirectory cannot be accessed!" -ErrorAction Stop
}

$scripts = Get-ChildItem $ScriptDirectory | Where-Object {$_.Extension -eq ".sql"}
$script_count = (Get-ChildItem $ScriptDirectory | Measure-Object).Count
$str_script = if($script_count -eq 1) {'script'} else {'scripts'}

Write-Host "Attempting to deploy $script_count $str_script to each server."

foreach ($server in $Servers) {
    Write-Host "Deploying to" $server.name

    foreach ($script in $scripts) {
        Write-Host "Installing" $script 
        Invoke-Sqlcmd -ServerInstance $server.Name -InputFile $script.FullName -DisableVariables

    }
}
{{< / highlight >}}

Step 3 done. 

As you can see, it's a pretty basic script. It grabs the list of servers from the config.json file, goes to the `ScriptDirectory` to loop through each file, and executes it on each SQL Server. I'm sure there are plenty of other ways to do this, but it works and it only needs to be run when there are changes to the maintenance scripts I want to deploy which isn't often.

The script is flexible for my needs. If I want to execute it for my restricted firewalled area, I just need to run:

{{< highlight powershell>}}
.\Deploy-MaintenanceScripts.ps1 -DeploySecure $true
{{< / highlight >}}

And it will install the scripts on every server in `zone2_servers`. 

If I want to deploy to a specific set of servers, I can run:

{{< highlight powershell>}}
.\Deploy-MaintenanceScripts.ps1 -DeploySecure $false -$ServerGroup zone1_servers
{{< / highlight >}}

And it will deploy to all servers with `zone1_servers` as their type. 

I've uploaded the code to [a repo in GitHub](https://github.com/tarynpratt/misc_sql_scripts/tree/master/DeployMaintenanceScripts) if anyone would find it useful. 

### Next Steps

Since I rarely need to make changes to the scripts, I don't have this currently setup in our CI/CD to deploy automatically when I push changes to GitHub. I could, and might at some point, then I wouldn't have to manually deploy via an automated script. 

While this works for me, what do you use to deploy maintenance scripts to all your SQL Servers? Do you have a homegrown process or a 3rd party tool? 
