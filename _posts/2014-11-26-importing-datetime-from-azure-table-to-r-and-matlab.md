---
layout: post
title: Importing DateTime from Azure Table to R and Matlab
comments: true
---

Processing time data from Azure Table in R and Matlab can be quite tricky. First DateTime gets converted to UTC when inserted into Azure Tables [[msdn](http://msdn.microsoft.com/en-us/library/azure/dd894027.aspx)]. Then Powershell's Export-CSV takes those dates and format them using the current locale (say "11/26/2014 5:31:16 AM" in North America). This is not what R and Matlab expect and conversions are needed to use those dates.

DateTime from Azure Table to R
---

R needs to be given the default .NET date format to parse the date string:

{% highlight R %}
> dateFromAzureTable <- "11/26/2014 5:31:16 AM"
> dateUtc <- as.POSIXct(dateFromAzureTable, tz="GMT", "%m/%d/%Y %I:%M:%S %p")
> dateUtc
[1] "2014-11-26 05:31:16 GMT"
{% endhighlight %}

The date can then be converted from UTC to local time:

{% highlight R %}
> dateLocal <- format(dateUtc, tz="", usetz=T)
> dateLocal
[1] "2014-11-25 21:31:16 PST"
{% endhighlight %}

DateTime from Azure Table to Matlab
---

Date parsing is similar in Matlab, using datenum():

{% highlight Matlab %}
>> dateFromAzureTable = '11/26/2014 5:31:16 AM'
>> dateUtc = datenum(dateFromAzureTable, 'mm/dd/yyyy HH:MM:SS AM'); datestr(dateUtc)
ans =
26-Nov-2014 05:31:16
{% endhighlight %}

The problem with Matlab is that until R2014a it did not support the notion of time zones [[mathworks](http://www.mathworks.com/help/matlab/dates-and-time-as-numeric-values.html)]. For older versions, Matlab's interop with Java gives access to the TimeZone class which provides the offset between UTC and local time:

{% highlight Matlab %}
>> offsetUtcToLocal = java.util.TimeZone.getDefault().getRawOffset() / (1000 * 60 * 60 * 24)
offsetUtcToLocal =
   -0.3333
>> dateLocal = dateUtc + offsetUtcToLocal; datestr(dateLocal)
ans =
25-Nov-2014 21:31:16
{% endhighlight %}

Further reading
---

This MSDN [article](http://msdn.microsoft.com/en-us/library/ms973825.aspx) has an enlightening list of common gotchas around processing dates.
