---
layout: post
title: Personal PDB Symbol Server
comments: true
---

Symbols of NuGet packages are typically found on [SymbolSource](http://www.symbolsource.org/). This works great until one creates binaries using C++, which SymbolSource [does not support yet](https://groups.google.com/forum/#!topic/symbolsource/VYDA7vpC6Nc). So instead I maintain a personal symbol server at [http://mmaitre314.blob.core.windows.net/symbols](http://mmaitre314.blob.core.windows.net/symbols). As usual, Visual Studio can be told to look up symbols there by adding this URL to the list of symbol file locations:

![SymbolsServerConfig]({{ site.url }}/images/SymbolsServerConfig.png)

The symbols also contain source information which the debugger can use to download source files from GitHub on the fly. To enable this, 'Enable Just My Code' needs to be unchecked and 'Enable source server support' checked.

![SourceServerConfig]({{ site.url }}/images/SourceServerConfig.png)