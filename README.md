[![NuGet](https://img.shields.io/nuget/v/Xam.Firebase.iOS.BitcodeStrip.svg)](https://www.nuget.org/packages/Xam.Firebase.iOS.BitcodeStrip/)

# Xam.Firebase.iOS.BitcodeStrip

Workaround for https://github.com/dotnet/macios/issues/22591.
Just add package reference to `Xam.Firebase.iOS.BitcodeStrip` whereever you reference `Xamarin.Firebase.iOS.*` to get the issue resolved.

```
dotnet add package Xam.Firebase.iOS.BitcodeStrip
```

## Synopsis

`bitcode_strip` shipped with xcode 16.4 expects to be executed from working directory it resides in, i.e. `/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin`.

If you run it from outside of its' directory it fails with `"ld: file cannot be open()ed, errno=2 path=strip"` error. It can not resolve `strip` utility located alongside.

`Xamarin.Firebase.iOS.*` binding packages wrap firebase sdk version that shipped with `bitcode` inside. `bitcode` is deprecated by Apple. Apple does not accept app store submissions with `bitcode` anymore. That is why bindings do rely on 'bitcode_strip' to remove bitcode from native frameworks. Strip invocation is done by extending msbuild process with following targets:

MacOS
```
<Target Name="_FirebaseStripBitcodeFromFrameworksOnMac">
    <!-- Get the frameworks to strip bitcode -->
    <FindInList 
        List="@(_FrameworkNativeReference)"
        ItemSpecToFind="%(_FrameworkNamesToStripBitcode.Identity)"
        MatchFileNameOnly="True">
        <Output TaskParameter="ItemFound" ItemName="_FrameworksToStripBitcode"/>
    </FindInList>

    <!-- Find the bitcode_strip command -->
    <Exec Command="xcrun -find bitcode_strip" ConsoleToMSBuild="true">
        <Output TaskParameter="ConsoleOutput" PropertyName="_BitcodeStripCommand" />
    </Exec>

    <!-- Strip the bitcode from frameworks -->
    <Exec Command="$(_BitcodeStripCommand) %(_FrameworksToStripBitcode.Identity) -r -o %(_FrameworksToStripBitcode.Identity)" />
</Target>
```
Windows
```
<Target Name="_FirebaseStripBitcodeFromFrameworksOnWindows"
        Condition="'$(IsMacEnabled)'=='true'">
    <!-- Get the frameworks to strip bitcode -->
    <FindInList 
        CaseSensitive="false"
        List="@(_FrameworkNativeReference)"
        ItemSpecToFind="%(_FrameworkNamesToStripBitcode.Identity)"
        MatchFileNameOnly="True">
        <Output TaskParameter="ItemFound" ItemName="_FrameworksToStripBitcode"/>
    </FindInList>

    <!-- Strip the bitcode from frameworks -->
    <Exec SessionId="$(BuildSessionId)"
            Command="&quot;%24(xcrun -find bitcode_strip)&quot; %(_FrameworksToStripBitcode.Identity) -r -o %(_FrameworksToStripBitcode.Identity)" />

    <CopyFileFromBuildServer 
        SessionId="$(BuildSessionId)" 
        File="%(_FrameworksToStripBitcode.Identity)" 
        TargetFile="%(_FrameworksToStripBitcode.Identity)" />
</Target>
```

So `bitcode_strip` is called by absolute path from current project directory and fails.

`Xamarin.Firebase.iOS.*` bindings are not maintained by microsoft anymore. The repository was archived on May 1, 2024 https://github.com/xamarin/GoogleApisForiOSComponents. You should consider moving away from it by making own bindings or use community ones out there if any. Meanwhile use this package as a workaround.

This package employs overrides of the targets mentioned above to ensure that working directory is properly set as expected:

MacOS
```
<!-- Capture bitcode_strip dir -->
<PropertyGroup>
    <_BitcodeStripDir>$([System.IO.Path]::GetDirectoryName($(_BitcodeStripCommand)))</_BitcodeStripDir>
</PropertyGroup>

<!-- Strip the bitcode from frameworks -->
<Exec WorkingDirectory="$(_BitcodeStripDir)" Command="./bitcode_strip %(_FrameworksToStripBitcode.Identity) -r -o %(_FrameworksToStripBitcode.Identity)"/>
```

Windows
```
<!--
    use subshell for safe change of working dir, so it will not cause any side effects on subsequent commands
    (cd "$(dirname "$(xcrun -find bitcode_strip)")" && ./bitcode_strip %(_FrameworksToStripBitcode.Identity) -r -o %(_FrameworksToStripBitcode.Identity))
-->
<Exec SessionId="$(BuildSessionId)"
        Command="(cd &quot;%24(dirname &quot;%24(xcrun -find bitcode_strip)&quot;)&quot; &amp;&amp; ./bitcode_strip %(_FrameworksToStripBitcode.Identity) -r -o %(_FrameworksToStripBitcode.Identity))" />
```

We rely on **determenistic order** of targets imports from nuget packages. The order is dictated by dependency graph. If package A references package B than targets from package B are imported before targets from package A. We do explicitly reference `Xamarin.Firebase.iOS.Core` package to ensure the overrides are imported after original targets. 

## Device Incremental Builds

Execution of the `bitcode_strip` target takes ten(s) of seconds on every device build, which slows down your incremental builds by that amount. During development, you generally don't care about bitcode until you submit to the App Store. Speed up incremental debug builds by disabling `bitcode_strip`.

```
  <PropertyGroup Condition=" '$(Configuration)' == 'Debug' ">
    <FirebaseStripBitcode>false</FirebaseStripBitcode>
  </PropertyGroup>
```

---
**Note**

Determenistic order is confirmed here https://github.com/NuGet/Home/issues/4229#issuecomment-271387190
> 4.x ensures everything is in dependency order and has the right behavior now.
___