---
title: "Unreal Engine Build Graph Copy issue"
datePublished: 2022-07-29T01:29:51.722Z
cuid: cl65setoi07ub0ynv8ddkbf0i
slug: unreal-engine-build-graph-copy-issue
tags: unreal-engine

---

Since there're not too much useful info in [Unreal Engine documentation](https://docs.unrealengine.com/4.27/en-US/ProductionPipelines/BuildTools/AutomationTool/BuildGraph/ScriptAnatomy/Tasks/), I have to dig this up by myself
```
ERROR: Source file 'C:\workspace\Binaries\Win64\UnrealEditor.target' is not under 'C:\workspace\UnrealEngine'
       while executing task <Copy Files="#InternalBinaries" From="C:\workspace\UnrealEngine" To="C:\workspace\UnrealEngine\LocalBuilds\ArchiveForUGS\Staging" Overwrite="True" ErrorIfNotFound="False" />
       at Engine/Build/Graph/Examples/CustomBuildEditorAndTools.xml(151)
       (see C:\workspace\UnrealEngine\Engine\Programs\AutomationTool\Saved\Logs\Log.txt for full exception trace)
```
So this is the line causing the error
```
<Tag Files="#OutputFiles" Except=".../Intermediate/..." With="#ArchiveFiles"/>
<Tag Files="#ArchiveFiles" Except="*.pdb" With="#ArchiveBinaries"/>
<Copy Files="#ArchiveBinaries" From="$(RootDir)" To="$(ArchiveStagingDir)"/>
```
The `#ArchiveBinaries` are simply all the filtered files output from the compiling. `$(RootDir)` is the engine folder (C:\workspace\UnrealEngine) and `$(ArchiveStagingDir)` is some target folder that doesn't matter. The copy should remain the relative strcutre here, saying `C:\workspace\path\to\file` should be copied to `$(ArchiveStagingDir)\path\to\file`

And the error is basically saying that some files is not under the `RootDir` and can't be copied with relative structure remaining, which makes sense, because you can't have something like `$(ArchiveStagingDir)\..\path\to\file`.

And the reason why we get files not under the `RootDir` is simple, because we don't put our game folder as a subfolder of Unreal Engine source. Once the reason is clear, the solution is simple and straight forward:
```
<Option Name="MyRootDir" DefaultValue="$(RootDir)\..\" Description ="The root dirctory contain both Engine and Game folders"/>
<Copy Files="#ArchiveBinaries" From="$(MyRootDir)" To="$(ArchiveStagingDir)"/>
```

What we learn today, Unreal want you to put your game under Engine folder if you use the source code, but you can choose to not listen to them :)