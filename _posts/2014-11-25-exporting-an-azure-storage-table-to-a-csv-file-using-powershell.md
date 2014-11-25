---
layout: post
title: Exporting an Azure Storage Table to a CSV file using PowerShell
comments: true
---

CSV files are staple food in data analysis. The Azure PowerShell does not include a way to export Storage Tables that way but that can be achieved by mixing a bit of PowerShell with a bit of .NET, reusing code from chriseyre2000's [query-azuretable2](https://github.com/chriseyre2000/Powershell/blob/master/Azure2/query-azuretable2.psm1).

First install the [Azure PowerShell](http://azure.microsoft.com/en-us/documentation/articles/install-configure-powershell/#Install) and open its command prompt. That loads all the Azure assemblies required in PowerShell.

Then add you account info

{% highlight PowerShell %}
Add-AzureAccount
{% endhighlight %}

and add a function to convert Azure Table Entities into something a bit more PowerShell-friendly

{% highlight PowerShell %}
function EntityToObject ($item)
{
    $p = new-object PSObject
    $p | Add-Member -Name ETag -TypeName string -Value $item.ETag -MemberType NoteProperty 
    $p | Add-Member -Name PartitionKey -TypeName string -Value $item.PartitionKey -MemberType NoteProperty
    $p | Add-Member -Name RowKey -TypeName string -Value $item.RowKey -MemberType NoteProperty
    $p | Add-Member -Name Timestamp -TypeName datetime -Value $item.Timestamp -MemberType NoteProperty

    $item.Properties.Keys | foreach { 
        $type = $item.Properties[$_].PropertyType;
        $value = $item.Properties[$_].PropertyAsObject; 
        Add-Member -InputObject $p -Name $_ -Value $value -TypeName $type -MemberType NoteProperty -Force 
    }
    $p
}
{% endhighlight %}

All that remains to do is connect to the Storage Table

{% highlight PowerShell %}
$storageAccount = [Microsoft.WindowsAzure.Storage.CloudStorageAccount]::Parse("DefaultEndpointsProtocol=https;AccountName=YourAccoutName;AccountKey=YourAccountKey")
$table = $storageAccount.CreateCloudTableClient().GetTableReference("YourTable")
$query = New-Object "Microsoft.WindowsAzure.Storage.Table.TableQuery"
$data = $table.ExecuteQuery($query)
{% endhighlight %}

and export the data

{% highlight PowerShell %}
$data |
    % { EntityToObject $_ } |
    Select-Object PartitionKey,Property0,Property1 |
    Export-Csv "data.csv" -NoTypeInformation
{% endhighlight %}
