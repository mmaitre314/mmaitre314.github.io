---
layout: post
title: Field Medic Custom Logging
comments: true
---

Field Medic is a Windows Phone app recording system events, which comes handy when debugging complex issues in Windows Phone apps. Besides a set of built-in logging profiles, it supports custom profiles targeted at more specific issues.

To install the app, search for 'Field Medic' in the Windows Phone Store or open [this link](http://www.windowsphone.com/en-us/store/app/field-medic/73c58570-d5a7-46f8-b1b2-2a90024fc29c). Then connect the Phone via USB and open the folder 'This PC\Windows Phone\Phone\FieldMedic\CustomProfiles' in Explorer, creating missing folders in that path as needed.

Field Medic uses [WPRP files](http://msdn.microsoft.com/en-us/library/windows/hardware/hh448223.aspx) to define custom profiles. [MultimediaMem.wprp]({{ site.url }}/download/MultimediaMem.wprp) and [MultimediaPerf.wprp]({{ site.url }}/download/MultimediaPerf.wprp) are two such files targeting the multimedia stack, the former focusing on memory usage and the latter on performance. Copy those two files to the 'CustomProfiles' folder on the Phone.

In Field Medic custom profiles are selected by going to the 'Advanced' page, opening the list of ETW providers, unselecting built-in categories, and selecting the appropriate profile at the bottom of the list under 'Custom Group'.

<img src="{{ site.url }}/images/2014-12-01-FieldMedic-Main.png" alt="Main" style="width: 200px;"/>|<img src="{{ site.url }}/images/2014-12-01-FieldMedic-Advanced.png" alt="Advanced" style="width: 200px;"/>|<img src="{{ site.url }}/images/2014-12-01-FieldMedic-Categories.png" alt="Categories" style="width: 200px;"/>

At that point Field Medic is ready for logging. A typical log session consists in:

- opening Field Medic and tapping 'Start Logging'
- opening the app being debugged and going through the repro steps
- coming back to Field Medic, tapping 'Stop Logging', entering a title, and hitting save

After reconnecting the Phone via USB the logs are available under 'This PC\Windows Phone\Phone\Documents\FieldMedic\reports'. They include .etl files which can be opened by [Windows Performance Analyzer](http://msdn.microsoft.com/en-us/library/windows/hardware/hh448170.aspx) (WPA) or any other tool processing .etl trace files.

![WPA]({{ site.url }}/images/2014-12-01-WPA.png)
