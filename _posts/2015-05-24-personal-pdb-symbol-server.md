---
layout: post
title: Personal PDB Symbol Server
comments: true
---

Symbols of NuGet packages are typically found on [SymbolSource](http://www.symbolsource.org/). This works great until one creates binaries using C++, which SymbolSource [does not support yet](https://groups.google.com/forum/#!topic/symbolsource/VYDA7vpC6Nc). So instead I maintain a personal symbol server at [http://mmaitre314.blob.core.windows.net/symbols](http://mmaitre314.blob.core.windows.net/symbols). As usual, Visual Studio can be told to look up symbols there by adding this URL to the list Symbol file locations:

![SymbolsServerConfig](http://matthieumaitre.info/images/SymbolsServerConfig.png)
