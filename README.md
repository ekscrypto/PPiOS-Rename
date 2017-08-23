PreEmptive Protection for iOS - Rename
======================================

*PreEmptive Protection for iOS - Rename*, or *PPiOS-Rename* for short, is a command line utility for obfuscating Objective-C class, protocol, property, and methods names, in iOS apps. It is a fork of [iOS-Class-Guard](https://github.com/Polidea/ios-class-guard) from [Polidea](https://www.polidea.com/), with extensive improvements and modifications.

*PPiOS-Rename* works by generating a special set of `#define` statements (e.g. `#define createArray y09FzL7T`) that automatically rename symbols during compilation. It includes a number of features:

* Analyze a Mach-O binary to identify symbols to be renamed
* Apply the renaming rules to the project source code
* Translate an obfuscated crash dump back to unobfuscated names

*PPiOS-Rename* works with more than just your project's code. It also automatically finds symbols to exclude from renaming by looking at all external/dependent frameworks and in Core Data (xcdatamodel) files. The renamed symbols will also be applied to your XIB/Storyboard files, and to any open-source CocoaPods libraries in your project.

[PreEmptive Solutions](https://www.preemptive.com/) also offers another product, [PreEmptive Protection for iOS - Control Flow](https://www.preemptive.com/products/ppios), that includes additional obfuscation transforms. *PPiOS-Rename* is meant to work alongside *PPiOS-ControlFlow*; together they provide much better protection than either one alone can provide.

*PPiOS-Rename* is licensed under the GNU GPL v2, but commercial support is also available from [PreEmptive Solutions](https://www.preemptive.com/contact/contactus) via a commercial support agreement. Please see LICENSE.txt for details.

> DEVELOPER NOTE: This fork includes a substantial rewrite of the git history, to fix [a corrupted commit in the original repo](https://github.com/nygard/class-dump/commit/509591f78f37905913ba0cbd832e5e4f7b925a8a). More details are in [the changelog](CHANGELOG.md).


How It Works
------------

*PPiOS-Rename* is designed to be used in two phases of your build and release process. In the first phase, *PPiOS-Rename* analyzes an unobfuscated compiled build of your app, to determine which symbols should be renamed and which should be excluded from renaming. In the second phase, *PPiOS-Rename* applies those renaming rules to the **source code** of your app, so that the next build made from that source will be obfuscated. These two phases can be integrated into your build and release processes in a number of ways, including back-to-back.

### Phase 1: Analyze
`ppios-rename --analyze [--symbols-map symbols.map] <Mach-O binary>`

In this phase, *PPiOS-Rename* analyzes the unobfuscated compiled build of your app to determine which symbols should be renamed. It parses all classes, properties, methods and i-vars defined in that file adding all symbols to a "rename list". Then it builds up an "excludes list" with standard reserved words, symbols from dependent frameworks, symbols in Core Data (xcdatamodel) files, and any symbols that have been explicitly excluded via command line arguments. It them combines those lists to generate a final list of symbols that will be renamed.

For each such symbol, it generates a random identifier, and writes a "map file" (`symbols.map`, by default) with the original names mapped to the new random names. That map file is the final output of the Analyze phase, and is a required input for the next phase, Obfuscate Sources.

> Note: Usually at this point, you should archive the `symbols.map` file. You will need it to be able to de-obfuscate any stack traces generated by builds that were obfuscated based on it.

### Phase 2: Obfuscate Sources
`ppios-rename --obfuscate-sources [--symbols-map symbols.map]`

In this phase, *PPiOS-Rename* reads in the map file and generates a header file (`symbols.h`, by default) that has `#define`s for each symbol to be renamed. It then finds the appropriate Precompiled Header (`.pch`) files in your source code and adds a `#include` with the path to the header file. Finally, it finds all XIBs/Storyboards in your source tree and directly updates the names inside those files.

Now, with the source modifications in place, you can build your app as usual. It will be compiled with the obfuscated symbols. (And any open-source CocoaPods will also have their symbols obfuscated.)

> Note: The Obfuscate Sources phase modifies the **source code** of your app, but you should not check in the changes it makes. If you do so, it will cause errors the next time you need to perform the Analyze phase, and will cause issues with Storyboards in the IDE. We recommend only using the Obfuscate Sources phase in your release (or automated) build process, and you should always clean/reset your source tree after the build, before doing any further development.


Supported Platforms
-------------------

*PPiOS-Rename* supports apps developed for:

* iOS 10, iOS 11
* iPhone, iPod touch, and iPad
* ARM 32-bit, ARM 64-bit, and x86 Simulator

Using:

* Xcode 8, Xcode 9
* OS X El Capitan, macOS Sierra
* Objective-C


Installation
------------

We suggest downloading one of the binary releases from the [Releases](https://github.com/preemptive/ppios-rename/releases) page. The archive contains a standalone binary that you can copy to an appropriate place on your system, e.g. `/usr/local/bin`. We suggest ensuring that the location is on your PATH. The release archive also includes other files such as this README, a changelog, and our license.


Project Setup
-------------

The basic process is:

1. Ensure all local source code changes have been committed
2. Build the program
3. Analyze the program
4. Apply renaming to the sources
5. Build the program again

For your first time using *PPiOS-Rename,* the following command (adjusted for your project) will perform the Analyze step:

    ppios-rename --analyze /path/to/program.app/program

The analyze process generates `symbols.map`, the file containing a symbol mapping that can be used to decode stack traces in the event of a crash. The symbols file created during a build that is released should always be archived for subsequent use.

Then the Apply Renaming step can be accomplished with the following:

    ppios-rename --obfuscate-sources

> Note: The Obfuscate Sources phase (invoked in the Apply Renaming step) modifies the **source code** of your app, but you should not check in the changes it makes. If you do so, it will cause errors the next time you need to perform the Analyze phase, and will cause issues with Storyboards in the IDE. We recommend only using the Obfuscate Sources phase in your release (or automated) build process, and you should always clean/reset your source tree after the build, before doing any further development.

Once you are comfortable using *PPiOS-Rename*, it can be easier to use if you integrate it into your Xcode project as part of the build process. This can be set up with the following process:

1. Open the project in Xcode.

2. Go to the Project Navigator, and select the project.

3. Select the icon to open the "Show project and targets list" (near the upper left corner of the main pane).

4. Select the target to obfuscate, right-click, and select Duplicate (Command-D).
> Note: Signing details for the target may need to be set in order for the build to complete successfully. These can be configured on the General tab.

5. <a name="configureAnalyze"></a>Select the duplicated target and rename it to `Build and Analyze <original-target-name>`.
> Note: If applying these changes to a framework project, you may need to use *underscores* instead of *spaces* in the new target names.

6. <a name="renameInfoPlist"></a>(Optional) Duplicating the target duplicates the associated `.plist` file with a default name. Rename the `.plist` file:

    1. Select Build Settings, select _All_ settings, select _Combined_ view, and search for `plist`.
    2. Update the value for `Info.plist File` to be consistent with that of the original target (something like `<project-name>/Build and Analyze <original-target-name>-Info.plist`).
    3. Move/rename the duplicated `.plist` file to the new name and path (e.g. using Finder).
    4. Delete the now stale reference to the default `.plist` file from the Project Navigator.

7. Select Build Phases.

8. Add a script phase by selecting the `+` (just above Target Dependencies), and then selecting New Run Script Phase (it should run as the last phase, and will by default).

9. Rename the phase from `Run Script` to `Analyze Binary`.

10. Expand the phase, and where it says `Type a script or ...`, paste the following script, adjusting for the correct path:

        PATH="$PATH:$HOME/Downloads/PPiOS-Rename-v1.2.0"
        [[ "$SDKROOT" == *iPhoneSimulator*.sdk* ]] && sdk="$SDKROOT"
        test -z "$sdk" && sdk="$CORRESPONDING_SIMULATOR_SDK_DIR"
        test -z "$sdk" && sdk="$CORRESPONDING_SIMULATOR_PLATFORM_DIR/Developer/SDKs/iPhoneSimulator${SDK_VERSION}.sdk"
        ppios-rename --analyze --sdk-root "$sdk" "$BUILT_PRODUCTS_DIR/$EXECUTABLE_PATH"

11. From the menu, select Product | Scheme | Manage Schemes.

12. If Autocreate Schemes is enabled, a new scheme for the duplicated target will have already been created. Rename it to `Build and Analyze <original-scheme-name>`, and close the dialog. Otherwise, create a new scheme for the Build and Analyze target.

13. <a name="configureRenaming"></a>Duplicate the original target again, and rename it to `Apply Renaming to <original-target-name>`. Again, optionally renaming the `Info.plist` file as described in [step 6](#renameInfoPlist).
> Notes: Signing details for the target may need to be set in order for the build to complete successfully. These can be configured on the General tab.
>
> If applying these changes to a framework project, you may need to use *underscores* instead of *spaces* in the new target names.

14. Delete all of the build phases in this target.

15. If there are any target dependencies, delete them as well.

16. Add a script phase, and rename it to `Apply Renaming to Sources` (this should be the only real action for this target).

17. Paste the following script, again adjusting for the correct path:

        PATH="$PATH:$HOME/Downloads/PPiOS-Rename-v1.2.0"
        ppios-rename --obfuscate-sources

18. <a name="renameApplyRenamingScheme"></a>Edit the scheme (or add one) for this new target, renaming the scheme to `Apply Renaming to <original-scheme-name>`.

19. These changes should be committed to source control at this point, since building the target to Apply Renaming will change the sources in ways that shouldn't generally be committed.

When ready to start testing an obfuscated build:

1. Ensure all local source code changes have been committed.

2. Build using the Build and Analyze scheme, producing the symbols file, `symbols.map`.

3. Commit or otherwise preserve the `symbols.map` file.

4. Build using the Apply Renaming scheme, which applies the renaming to the sources.

5. Build using the original scheme.

6. Revert changes to the sources before continuing development.

Once renaming has been applied to the sources, the process of building and testing for different destinations can be repeated using the original scheme (step #5), as long as you haven't reverted the sources yet (step #6).

If you modify the original build target or scheme, be sure to delete and recreate the Build and Analyze target as above. Under certain conditions, the Apply Renaming target and scheme will need to be recreated as well.


Using PPiOS-Rename with PPiOS-ControlFlow
-----------------------------------------

*PreEmptive Protection for iOS - Rename* (*PPiOS-Rename)* provides the "renaming" obfuscation, which is the most-common type of obfuscation typically applied to applications to help protect them from reverse engineering, intellectual property theft, software piracy, tampering, and data loss. There are additional obfuscation techniques, however, that are critically important for serious protection of apps. [PreEmptive Solutions](https://www.preemptive.com/) offers another product, [PreEmptive Protection for iOS - Control Flow](https://www.preemptive.com/products/ppios), that includes additional obfuscation transforms. *PPiOS-Rename* is meant to work alongside *PPiOS-ControlFlow*; together they provide much better protection than either one alone can provide.

Simple instructions for using them together are available in the documentation for *PPiOS-ControlFlow*.


Demonstration
-------------

Below is a demonstration of the effects of applying obfuscation. The optimized binary was reverse engineered using [Hopper](http://www.hopperapp.com/) with no debugging symbols. This is a realistic example of what an attacker would see using reverse engineering tools.

Original code:

<img width="350" alt="original-sized" src="https://raw.githubusercontent.com/preemptive/PPiOS-Rename/master/images/original-sized.png">

Reverse engineered code: (what an attacker would see)

<img width="350" alt="unobfuscated-sized" src="https://raw.githubusercontent.com/preemptive/PPiOS-Rename/master/images/unobfuscated-sized.png">

Reverse engineered code with PPiOS-Rename:

<img width="350" alt="renamed-sized" src="https://raw.githubusercontent.com/preemptive/PPiOS-Rename/master/images/renamed-sized.png">

Reverse engineered code with both *PPiOS-Rename* and *PPiOS-ControlFlow*:

<img width="350" alt="controlflow-sized" src="https://raw.githubusercontent.com/preemptive/PPiOS-Rename/master/images/controlflow-sized.png">

As seen, the code is relatively straightforward to understand with no obfuscation. It's not obvious after applying *PPiOS-Rename* obfuscation, but the logic could still be inferred by the system framework methods being used. And finally, it's extremely difficult to understand the logic in the last version with *PPiOS-ControlFlow* obfuscation. The decompiled code was actually significantly longer than shown here.


Sample Project
--------------

A sample project demonstrating the process of obfuscating an iOS app is available on GitHub: [PPiOS-Sample-Vie](https://github.com/preemptive/PPiOS-Sample-Vie). This project uses both *PPiOS-ControlFlow* and *PPiOS-Rename*, but can be used to examine their effects independently. All of the steps required to integrate *PPiOS* into an existing Xcode project have already been applied. The configuration has also been adjusted to ensure a positive user experience with the obfuscated app. Documentation for the project discusses in detail how to build and use the sample, how the project was configured, and how to interpret the build output.


Troubleshooting
---------------

### Missing `.pch` file
During the Obfuscate Sources phase, you may get an error:

    Error: could not find any *-Prefix.pch files under .

This is because *PPiOS-Rename* is attempting to add an `#include` to a Precompiled Header file, and it can't find a suitable file to add it to. This is typically because projects created in Xcode 6 and above don't contain a `.pch` file by default.

To fix this, add a `.pch` file as follows:

1. In Xcode go to *File -> New -> File -> iOS -> Other -> PCH File*.

2. Name the file e.g. `MyProject-Prefix.pch`. *PPiOS-Rename* looks for a file matching `*-Prefix.pch`.

3. At the target's *Build Settings*, in *Apple LLVM - Language* section, set **Prefix Header** to your PCH file name.

4. At the target's *Build Settings*, in *Apple LLVM - Language* section, set **Precompile Prefix Header** to *YES*.

### Error: Fat file does not contain a valid Mach-O binary for the specified architecture (...). If you are trying to obfuscate a static library, please review the 'Obfuscating Static Libraries' section of the documentation.

You have tried to run analysis on a static library or framework.

If you are trying to analyze a static library, please follow the instructions in the [Obfuscating Static Libraries](#obfuscating-static-libraries) section below.

If you are trying to analyze a framework, sometimes it will work if you `--analyze` the `AppName.framework` directory created by Xcode when archiving. Try archiving the framework from Xcode and use the `AppName.framework` folder created inside the project's derived data folder (`~Library/Developer/Xcode/DerivedData/ProjectName-.../...`).

### Undefined symbols / exclusions

During the build, after the Obfuscate Source phase, you may see errors like this:

    Undefined symbols for architecture i386:
      "_OBJC_CLASS_$_n9z", referenced from:
          objc-class-ref in GRAppDelegate.o

You might also see `unresolved external` linker errors, e.g. if you used a C function and named an Objective-C method using the same name.

These errors usually mean that *PPiOS-Rename* obfuscated a symbol that needs to be excluded for some reason. You can find the symbol by searching `symbols.map` or `symbols.h` for the referenced symbol (`n9z`, in this example) to see what the original name was. Then you can exclude the symbol via command line arguments to the Analyze phase, via `-F` or `-x`, described below.

In this example, if `n9z` had mapped to `PSSomeClass`, you would add `-F '!PSSomeClass'` to your arguments when running `--analyze`.

#### Filter Classes, Protocols, and/or Categories
The `-F` option defines a filter against which class, protocol, and category names will be matched. The argument to `-F` is a glob pattern supporting `*` (any number of any character) and `?` (any single character). If the first character of the pattern is a `!` then the filter will _exclude_ any matching classes, protocols, and categories. If the first character is _not_ a `!`, then the filter will _include_ any matching classes, protocols, and categories.

The default filter is equivalent to `-F '*'` and the system behaves as if it is always the first filter specified. Additional filters can be specified on the command line, and each one overrides the rules from the ones that came before. For example:

    -F '!A?H*' -F 'ATH*'

This will filter out all classes, protocols, and categories that start with an "A", have any next character, then have an "H", **except** for classes, protocols, and categories that specifically start with "ATH". All other classes will be "filtered in" by the default rule.

Filter patterns are case sensitive, so `-F ABC` will match differently than `-F abc`. There is basic support for character classes, so you can match either with e.g. `-F '[Aa][Bb][Cc]'`.

#### Exclusion propagation
When excluding items via `-F`, if the excluded item matches a class, protocol, or category name, then additional exclusions may be applied based on that name.

For example, if a class name is excluded, then the following will also be excluded (assuming "ClassName" is the class name):

1. ClassNameProtocol
2. ClassNameDelegate
3. All of the methods and properties defined within the class

Also see the section below about property name exclusions.

#### Symbol filter
You can exclude specific symbols by using the `-x` argument in the Analyze phase. For example:

    -x 'deflate' -x 'curl_*'

This will exclude symbols named *deflate* and symbols that start with *curl_*.

`?` matches any single character, while `*` matches any number of characters.

> Note: Symbols excluded with `-x` will be excluded regardless of positive `-F` filters (`-x` "wins"). However, `-x` exclusions do not propagate like `-F` inclusions/exclusions do. For example, specifying `-F '!*' -F MyClass -x MyClass` will not rename the `MyClass` class itself, but will rename the properties and methods contained therein.

#### Property name exclusions
When excluding properties (either via `-x` or via propagation from `-F`), the following names are also excluded (assuming the propery name is `propertyName`):

1. _propertyName
2. setPropertyName
3. isPropertyName
4. _isPropertyName
5. setIsPropertyName

### Issues using an obfuscated framework

Ensure you have excluded all the classes, properties, etc specified in the distributed header files as users of the framework may encounter linking and/or runtime issues if some were renamed.

#### Undefined symbols for architecture ...: "_OBJC_CLASS_$_SomeClass", referenced from: objc-class-ref in ObjectFile.o

This linking issue occurred because `SomeClass` was not excluded when analyzing the framework. Add `-F '!ClassName'` to the command line when analyzing. You will then need to obfuscate, build, and redistribute the framework.

#### 'NSInvalidArgumentException', reason: '-[_ClassName setPropertyName:]: unrecognized selector sent to instance 0x145a7c90'

This end user application runtime failure occurred because `PropertyName` was not excluded when analyzing the framework. Add `-x PropertyName` to the the command line when analyzing. You will then need to obfuscate, build, and redistribute the framework.

### XIB and Storyboards limitations
If you're using external libraries which provide Interface Builder files, be sure to ignore those symbols as they won't work when you launch the app and try to use them. You can do that using the `-F` option to the Analyze phase.

### Key-Value Observing (KVO)
It is possible that during obfuscation KVO will stop working. Most developers use hardcoded strings to specify *KeyPath*.

``` objc
- (void)registerObserver {
    [self.otherObject addObserver:self
                       forKeyPath:@"isFinished"
                          options:NSKeyValueObservingOptionNew
                          context:nil];
}

- (void)unregisterObserver {
    [otherObject removeObserver:self
                     forKeyPath:@"isFinished"
                        context:nil];
}

- (void)observeValueForKeyPath:(NSString *)keyPath
              ofObject:(id)object
                change:(NSDictionary *)change
               context:(void *)context
{
  if ([keyPath isEqualToString:@"isFinished"]) {
    // ...
  }
}
```

This will not work. The property *isFinished* will get a new name and the hardcoded string will not reflect the change.

Remove any *keyPath* and change it to `NSStringFromSelector(@selector(keyPath))`.

**The fixed code should look like this:**

``` objc
- (void)registerObserver {
    [self.otherObject addObserver:self
                       forKeyPath:NSStringFromSelector(@selector(isFinished))
                          options:NSKeyValueObservingOptionNew
                          context:nil];
}

- (void)unregisterObserver {
    [otherObject removeObserver:self
                     forKeyPath:NSStringFromSelector(@selector(isFinished))
                        context:nil];
}

- (void)observeValueForKeyPath:(NSString *)keyPath
              ofObject:(id)object
                change:(NSDictionary *)change
               context:(void *)context
{
  if ([keyPath isEqualToString:NSStringFromSelector(@selector(isFinished))]) {
    // ...
  }
}
```

### Serialization
If you use serialization (e.g. `NSCoding` or `NSUserDefaults`), affected classes will have to be excluded from obfuscation. If you don't, then you won't be able to generate new symbols (i.e. the Analyze phase) without breaking deserialization of existing data.

### "Double obfuscation detected" error
This error happens when `--obfuscate-sources` is used on the same source tree twice. This can result in your application not being obfuscated. Make sure that the source tree is always reset to an unmodified state before using `--obfuscate-sources`.

### "Analyzing an already obfuscated binary" error
This error happens when `--analyze` is used on an already obfuscated binary. This can result in your application not being obfuscated. Make sure that your program is always rebuilt from clean and non-obfuscated source code before attempting to run the analysis process.


Advanced Topics
---------------

### Locating the Binary and dSYM Files.

When looking to verify obfuscation or send de-obfuscated dSYMs to analytics services, you must first locate the binary and dSYM files.

#### Xcode Archive

If you created an archive, Xcode would have placed it in the *archives* directory, `~/Library/Developer/Xcode/Archives/{Date}/{AppName} {Date}, {Time}.xcarchive`. Inside there, you should find:

  * Binary: `Products/Applications/{AppName}.app/{AppName}`
  * dSYM: `dSYMs/{AppName}.app.dSYM
  >Note: If you have uploaded your archive to Apple's App Store it may have been recompiled from bitcode and you would need to download the new dSYM files from either <https://itunesconnect.apple.com> or by using the *Download dSYMS...* button in Xcode's *Organizer* window. The *Download dSYMS...* button will download a dSYM for each architecture to the same `dSYMs` in the *archives* directory, but it will be named `{SOME GUID}.dSYM`.

#### .ipa File

If you have an `{AppName}.ipa` file, you will need to extract it by running `unzip {AppName}.ipa`. Inside the `Payload` directory, you should find:

  * Binary: `{AppName}.app/{AppName}`
  * dSYM: Is **not** included in the `.ipa` file.

#### Command Line Build

If you build from the command line (e.g. `xcodebuild`), this will typically create a `build` directory. Inside the `build` directory, you should find:

   * Binary: `Release-[iphoneos|iphonesimulator]/{AppName}.app/{AppName}`
   * dSYM: `Release-[iphoneos|iphonesimulator]/{AppName}.app.dSYM`

### Verifying obfuscation

To verify that your app has been obfuscated, use the `nm` utility, which is included in the Xcode Developer Tools. Run:

    nm path/to/your/binary | less

This will show the symbols from your app. If you do this with an unobfuscated build, you will see the original symbols. If you do this with an obfuscated build, you will see obfuscated symbols.

>Note: `nm` will not work properly after stripping symbols from your binary. You can use the `otool` utility if you need to check for the Objective-C symbols after stripping.

`otool` will show unneeded information, but it can be filtered using `grep` and `awk` to only show symbols:

    otool -o /path/to/your/binary | grep 'name 0x' | awk '{print $3}' | sort | uniq

>Note: The `X__PPIOS_DOUBLE_OBFUSCATION_GUARD__` symbol is used to help ensure renaming is applied properly.


### Reversing obfuscation in crash dumps

*PPiOS-Rename* lets you reverse the process of obfuscation for crash dump files. This is important so you can find the original classes and methods involved in a crash. It does this by using the information from a map file (e.g. `symbols.map`) to modify the crash dump text, replacing the obfuscated symbols with the original names. For example:

    ppios-rename --translate-crashdump --symbols-map path/to/symbols_x.y.z.map path/to/crashdump path/to/output

### Reversing obfuscation in dSYMs

It is possible to reverse the process of obfuscation in dSYMs by using a utility included with [PPiOS-ControlFlow](https://www.preemptive.com/products/ppios). The de-obfuscated dSYMs let you see the original names in automatic crash reporting tools such as HockeyApp, Crashlytics, Fabric, BugSense/Splunk Mint, or Crittercism. It does this by using the information from a map file (e.g. `symbols.map`) to generate a "reverse dSYM" file that has the non-obfuscated symbol names in it. For example:

    /usr/local/share/preemptive/PPiOS/bin/llvm-dsymutil -ppios-map=path/to/symbols_x.y.z.map path/to/input.dSYM -o=path/to/output.dSYM

>Note: If you do not specify an output dSYM by passing the `-o` argument, the input dSYM **itself** will be altered by running the `llvm-dsymutil` command.

The resulting dSYM file can be uploaded to e.g. HockeyApp.

dSYMs use a binary format that must be interpreted correctly by `llvm-dsymutil`. Thus the version of `llvm-dsymutil` from *PPiOS-ControlFlow* must match the major version of Xcode used to produce the dSYMs. For example, `llvm-dsymutil` from *PPiOS-ControlFlow* version 2.5 can only be used with Xcode 8, the latest version of Xcode supported by this version of *PPiOS-ControlFlow*.

Until Xcode 9 is supported by *PPiOS-ControlFlow*, dSYM translation is unavailable for apps or libraries built with Xcode 9.

> DEVELOPER NOTE: The original `--translate-dsym` logic did not work properly with small symbol names nor when the obfuscated names were a different size than the originals. It has been replaced by a feature in *PPiOS-ControlFlow*.

### Analyzing Dynamic Frameworks

Analyzing a dynamic framework is similar to analyzing an application, but will probably require many more filters. To start, use:

    ppios-rename --analyze /path/to/ProjectName.framework/ProjectName

Instead of seeing `Processing external symbols from ProjectName...`, which you will see for other frameworks, you should see `Processing internal symbols from ProjectName...`

If the `--analyze` is not properly finding your framework, then add the `--framework` argument:

    ppios-rename --analyze --framework ProjectName /path/to/framework/executable

You then need to determine and use the proper filters. You will need to choose one of two ways depending on how many public APIs you include in your framework:

* Exclude everything which is mentioned in the `.h` files you distribute with the framework.
* Start with a `-F '!*'` (exclude everything) and then include things not mentioned in those headers.

 Once you have run `ppios-rename --analyze` with the proper filters, continue with the standard `ppios-rename --obfuscate-sources` step and follow the rest of the instructions.

### Obfuscating Static Libraries

Static libraries cannot be directly processed by *PPiOS-Rename*, but it is possible to work around the technical issue preventing direct processing. Although the initial setup is somewhat involved, once this is complete the build process is no more complicated than with *PPiOS-Rename* other projects. The basic idea is:

1. Create a workspace to simplify interactions with the library.
2. Create an app that uses the static library.
3. Use the app for the analysis portion of *PPiOS-Rename*.
4. Use the analysis to apply renaming to the static library's sources.
5. Build the static library.

Importing the static library into the wrapping app pulls all of these classes and all of the classes that they reference into the app. Using this as the basis for analysis would rename **all of the symbols in the static library**. The resulting library would be unusable because all of the identifiers in the API would be renamed. In order for the static library to be usable externally, all of the public symbols need to be excluded from renaming. Excluding these classes **must be done manually**. `ppios-rename` will exclude all of the members of these classes, automatically, once the classes are excluded.

The procedure is as follows:

1. If your static library project (referred to here as `StaticLib`) does not already have a containing workspace, create one:
    1. In Xcode, go to `File` > `New` > `Workspace ...`.
    2. Choose an appropriate name for the workspace (referred to here as `LibWorkspace`), and save it in the directory containing the `StaticLib` project directory. See the source tree layout shown below.
    3. Go to `File` > `Add Files to "LibWorkspace"...`.
    4. Add the project file `StaticLib/StaticLib.xcodeproj`.

2. From within the workspace, create a new project:
    1. In Xcode, go to `File` > `New` > `Project ...`.
    2. Select `iOS`, and under the `Application` section select `Single View Application`.
    3. When choosing the options, specify the `Product Name` as `WrappingApp`, and be sure to select Objective-C as the language.
    4. Specify the directory to store the `WrappingApp` project as a sibling of the static library's project directory. These instructions expect a source tree layout like the following, with everything under `someParentDir` under source control:

            someParentDir/StaticLib/StaticLib.xcodeproj
            someParentDir/LibWorkspace.xcworkspace
            someParentDir/WrappingApp/WrappingApp.xcodeproj

    5. At the bottom of this dialog, specify `Add to:` the `LibWorkspace` workspace, and select `Create`.
    6. Select the WrappingApp target, then clean and build the project to verify that the project builds.
    7. Go to the Project Navigator, select the `WrappingApp` project, and select the `WrappingApp` target.
    8. Select `General`, scroll to the bottom, and select the `+` to add to the list of `Linked Frameworks and Libraries`.
    9. Add `libStaticLib.a`. This should appear under the `Workspace` section in the dialog. You may need to close and reopen `LibWorkspace` after adding `StaticLib` in order to get it to appear correctly.
    10. Select `Build Settings`, search for `other linker`, and set `Other Linker Flags` to include `-ObjC`.
    11. Clean and build again to verify that the workspace builds correctly. It should build the static library first and then the app.

3. Setup *PPiOS-Rename* to analyze the app:
    1. Follow instructions [5-12 in `Project Setup` above](#configureAnalyze), but apply them to `WrappingApp`, rather than the static library, and modify the original `WrappingApp` target. Duplication of the target is unnecessary.
    2. You may need to close and re-open the workspace to get Xcode to use the correct target name in the report navigator.
    3. <a name="countUniqueSymbols"></a>Clean and build the `Build and Analyze WrappingApp` target.
    4. Review the build log in the report navigator and look for the number of `Generated unique symbols`. This number should include all of the public and all of the non-public symbols from `StaticLib`. This will also include a small number of symbols from classes in the app itself. These should be benign, but can be excluded manually if necessary.
    5. Create a list of public types for the static library named `public-types.list`. Either create the list using the following procedure, or create the list manually. The procedure requires that the names of the public header files match the names of the types, or follow `AffectedType+CategoryName.h` convention.
        1. Go to the Report Navigator, review the log for a build of `StaticLib`, select a copy file task for one of the header files, right-click and `Copy`.
        2. Paste the result in a text editor.
        3. Select and copy the build output path. This should have the form: `<products-directory>/include/StaticLib`, and contain the string `/DerivedData/`.
        4. At a command line, run the following commands, from `someParentDir`:

                # Replace ".../include/StaticLib" with the path
                # Replace "StaticLib" with the name of the umbrella header
                ls ".../include/StaticLib" | sed 's/.*+//' | sed 's/[.]h$//' | grep -v StaticLib > public-types.list
                # Or omit the grep -v part if you have no umbrella header
                ls ".../include/StaticLib" | sed 's/.*+//' | sed 's/[.]h$//' > public-types.list
                # verify the contents
                head public-types.list
                # SLAPublicClass
                # SLProtocolForSomething
                # SLSomeCategory
                # ...

    >Note: This is an additional point of maintenance: as types are added to or removed from the public API of the library, the `public-types.list` will need to be updated accordingly.

    6. Replace the analyze script (`Analyze Binary` run script phase) with the following to exclude the public types from renaming:

            PATH="$PATH:$HOME/Downloads/PPiOS-Rename-v1.2.0"
            [[ "$SDKROOT" == *iPhoneSimulator*.sdk* ]] && sdk="$SDKROOT"
            test -z "$sdk" && sdk="$CORRESPONDING_SIMULATOR_SDK_DIR"
            test -z "$sdk" && sdk="$CORRESPONDING_SIMULATOR_PLATFORM_DIR/Developer/SDKs/iPhoneSimulator${SDK_VERSION}.sdk"
            ppios-rename --analyze --sdk-root "$sdk" \
                $(for each in $(cat ../public-types.list) ; do printf -- '-F !%s ' "$each" ; done) \
                "$BUILT_PRODUCTS_DIR/$EXECUTABLE_PATH"

    7. Repeat steps [3.iii - 3.iv](#countUniqueSymbols). The number of unique symbols should decrease, but still be significant. This should be the count of all of the non-public symbols (non-public symbols for which there are not public symbols with the same name).
    8. Review the list of types that will be renamed by executing the following at a command line (assuming all of the type names are prefixed with the two letters `SL`):

            cat WrappingApp/symbols.map | awk '{print $3}' | sed 's/[",]//g' | grep '^SL' > renamed-types.txt

4. Modify the static library project to apply the renaming:
    1. Follow instructions [13-16 in `Project Setup` above](#configureRenaming), applying them to the static library target.
    2. The call to `ppios-rename` needs to reference the `symbols.map` file from the WrappingApp project, using the `--symbols-map` option. Use this script for the new Run Script phase (adjusting the path as necessary):

            PATH="$PATH:$HOME/Downloads/PPiOS-Rename-v1.2.0"
            ppios-rename --obfuscate-sources --symbols-map ../WrappingApp/symbols.map

    3. Follow instruction [18 in `Project Setup` above](#renameApplyRenamingScheme).

5. All of these changes (to the static library, WrappingApp, and `public-types.list`) should be committed to source control at this point, since building the target to Apply Renaming will change the sources in ways that shouldn't generally be committed.

The process to build the static library with renaming becomes:
1. Ensure that build environment is clean (e.g. all stale symbols files have been removed, and modifications to .xib files have been reverted).
2. Clean and build the `Build and Analyze WrappingApp` target, which should also clean and build the static library.
3. Build the `Apply Renaming to StaticLib` target.
4. Clean and build the the `StaticLib` target.


Command Line Argument Reference
-------------------------------

```
ppios-rename --analyze [options] <mach-o-file>
  Analyze a Mach-O binary and generate a symbol map

  Options:
    --symbols-map <symbols.map>   Path to symbol map file
    -F '[!]<pattern>'             Filter classes/protocols/categories
    -x '<pattern>'                Exclude arbitrary symbols
    --arch <arch>                 Specify architecture from universal binary
    --sdk-root <path>             Specify full SDK root path
    --sdk-ios <version>           Specify iOS SDK by version
    --framework <name>            Override the detected framework name

ppios-rename --obfuscate-sources [options]
  Alter source code (relative to current working directory), renaming based on the symbol map

  Options:
    --symbols-map <symbols.map>   Path to symbol map file
    --storyboards <path>          Alternate path for XIBs and storyboards
    --symbols-header <symbols.h>  Path to obfuscated symbol header file

ppios-rename --translate-crashdump [options] <input crash dump file> <output crash dump file>
  Translate symbolicated crash dump

  Options:
    --symbols-map <symbols.map>   Path to symbol map file

ppios-rename --list-arches <mach-o-file>
  List architectures available in a fat binary

ppios-rename --version
  Print out version information

ppios-rename --help
  Print out usage information
```

---------------------------------------------------------------------
Copyright 2016-2017 PreEmptive Solutions, LLC
