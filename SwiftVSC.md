# **iOS Development is Visual Studio Code**

[Visual studio code description here and LSP server].

Since release of SourceKit-LSP tool it become possible to integrate fully-fledged swift language support through LSP protocol. Unfortunately, 
it's still not there for iOS specific development, as SourceKit-LSP heavily relies on SwiftPM, which doesn't yet have support for iOS binaries.
But fear not - with some additional setup we can configure VSC to enable language features for Swift with iOS specific libs, along with bulding, testing
and debugging in simulator.

## **Tools Setup**

### **VSCode Insiders build (if you want to use experimental features) or regular VSCode**
  
  Download and istall : https://code.visualstudio.com/insiders/, or https://code.visualstudio.com

### **XCode & Toolchain snapshot (matching sourcekit-lsp version)**
  Depending on which version of sourcekit-lsp you'd like to use you have to select right swift toolchain:
  https://swift.org/download/#snapshots

  If you choose to use experimental branch, currently used snapshot is https://swift.org/builds/development/xcode/swift-DEVELOPMENT-SNAPSHOT-2020-05-24-a/swift-DEVELOPMENT-SNAPSHOT-2020-05-24-a-osx.pkg. Although it can change rapidly, so please check each time when istalling with branch. 
  
### **SourceKit-LSP:** 

  sourcekit-lsp is included in Swift Toolchain package, although if you want to use experimental features, I suggest you build it from this branch:
  (build from https://github.com/prostakm/sourcekit-lsp/tree/semantic_tokens to enable semantic tokens)

   **Building from branch**

  Checkout branch using git 
  ```
  git clone -b semantic_tokens git@github.com:prostakm/sourcekit-lsp.git
  ``` 

  Navigate to root dir and run `swift build` command : 

  ```sh
  $ export TOOLCHAINS=swift
  $ swift package update
  $ swift build
  ```
  You will find `sourcekit-lsp` executable in `.build/debug/sourcekit-lsp` directory or similar

### **SourceKit-LSP plugin for VSCode**

  again, either select one from plugin marketplace in VSCode, or if you want to use experimental features, please build one from a branch provided here: https://github.com/prostakm/sourcekit-lsp/tree/semantic_tokens

  **Prerequisite**: To build the extension, you will need Node.js and npm: https://www.npmjs.com/get-npm.

  Clone repo as above and navigate to `Editors/vscode` and run `npm run createDevPackage` : 

  ```sh
  $ cd Editors/vscode
  $ npm run createDevPackage
  ```

  resulting .vsix file would be placed in /out directory - install it through VSCode using the `Extensions > Install from VSIX...` command from the command palette.

### **LLDB debugger plugin for VSCode**

  https://github.com/vadimcn/vscode-lldb, or install from VSCode plugin marketplace

  Set lldb library from toolchain you'd like yo use - you can do it in extension settings or in settings.json file, f.e.: 

  `"lldb.library": "/Applications/Xcode.app/Contents/SharedFrameworks/LLDB.framework/Versions/A/LLDB"` 

### **Xcodegen**

  We will use XCodegen to conviniently handle Xcode project files, please follow install instructions here: https://github.com/yonaskolb/XcodeGen. Add `project.yml` file with settings matching those in `Package.swift` file, f.e.

  ```yml
  name: [PROJECT_NAME]
options:
  bundleIdPrefix: com.mp
packages:
  Yams:
    url: https://github.com/jpsim/Yams
    from: 2.0.0
targets:
  TempContacts:
    type: application
    platform: iOS
    deploymentTarget: "13.4"
    sources:
      - [SOURCE_DIRECTORY]
  ```
  

## Configuring SwiftPM file

Create a swift package : 

` swift package init --type library `

Define targets, and sources as follows: 

```swift
import PackageDescription

let package = Package(
    name: "[PACKAGE_NAME]",
    platforms: [ .iOS(.v13)],
    dependencies: [],
    targets: [
        .target(
            name: "[PRODUCT_NAME]",
            dependencies: [],
            path: ".",
            sources: ["[SOURCE_DIR]"])
    ]
)
```


  
Building project using swift pm 

```swift
swift build -Xswiftc "-sdk" -Xswiftc \
(xcrun --sdk iphonesimulator --show-sdk-path) \
-Xswiftc "-target" -Xswiftc "x86_64-apple-ios13.4-simulator" 
```

Above command allow us to build sources specified in Package.swift file, create intermediate files and index files used by sourcekit-lsp for many features

## Configuring lsp extension - workspace, target , sdk

After installing sourcekit-lsp plugin we have to configure it specifically for iOS project, 
we have to pass similar arguments to it as when we build project using swiftPM. Go to `settings.json` (or extension settings) and configure SourceKit-LSP plugin with same arguments used when building project: 

```json
"sourcekit-lsp.serverPath": "/Users/maciekp/Library/Developer/Xcode/DerivedData/sourcekit-lsp-djxwdlfgidroldfjyqyhunsujdmc/Build/Products/Debug/sourcekit-lsp", //Path to sourcekit-lsp executable
"sourcekit-lsp.toolchainPath": "/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS13.4.sdk", //toolchain path
"sourcekit-lsp.serverArguments": [
      "-Xswiftc",
      "-sdk",
      "-Xswiftc",
      "/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator13.4.sdk", //Path to SDK
      "-Xswiftc",
      "-target",
      "-Xswiftc",
      "x86_64-apple-ios13.4-simulator" //Build target
    ]
```


## Enabling semantic highlighting

If you decided to use experimental features like semantic highlighting - add also this line for `settings.json`

- `"editor.semanticHighlighting.enabled": true,`

## Adding scripts, connecting XCodegen, Build script 

Although I think It would be possible to configure all necessary build scripts for iOS development using swiftc and swift compiler only, It's much more convenient to use xcodebuild and xcodeprojects now. Since maintaining Xcode projects without Xcode is a real pain, we will use awesome tool called Xcodegen, which allows us to generate xcodeproj based on simple .yaml files. 

Let's now connect each tool with few defined tasks as follow. 

First let's start with project file generation, this task will generate an `.xcodeproj` based on `project.yaml` file, it will use cache so if nothing changes it won't regenerate any files. 

```json
{
  "label": "Gen XCodeproj", //label
  "type": "shell",
  "command": "xcodegen generate --use-cache"
}
  ```

Next let's create a task for building an app using `xcodebuild` and project file generated in previous step:

```json
{ // Building an app 
  "label": "Build ios",
  "type": "shell",
  "command": "xcodebuild -scheme 'TempContacts' -derivedDataPath build -configuration Debug -project TempContacts.xcodeproj -destination 'platform=iOS Simulator,name=iPhone 11' build",
  "dependsOn": [
    "Gen XCodeproj"
  ],
  "group": {
    "kind": "build",
    "isDefault": true
  }
}
```
Explanation : 
- `-scheme 'TempContacts'` scheme name, as in `project.yaml` file
- `-project TempContacts.xcodeproj` generated project file name
- `-destination 'platform=iOS Simulator,name=iPhone 11'` build target destination
- `"dependsOn": [
    "Gen XCodeproj"
  ]` guarantees regenerating project file before each build, in case of changes

Next we need to set up simulator task, quite self explanatory - just open an simulator app bundled in Xcode package:

```json
{ // running simulator
  "label": "runSimulator",
  "type": "shell",
  "command": "open /Applications/Xcode.app/Contents/Developer/Applications/Simulator.app/"
}
```
When we have this set up let's configure app running tasks: 

```json
{ // Running an app
      "label": "Run ios",
      "type": "shell",
      "dependsOn": [
        "Build ios",
        "runSimulator"
      ],
      "command": "xcrun simctl install booted build/Build/Products/Debug-iphonesimulator/TempContacts.app && xcrun simctl launch booted com.mp.TempContacts",
      "group": {
        "kind": "build",
        "isDefault": true
      },
      "problemMatcher": []
    },
    { // Launch app and wait for debugger
      "label": "Debug ios",
      "type": "shell",
      "dependsOn": [
        "Build ios",
        "runSimulator"
      ],
      "command": "xcrun simctl install booted build/Build/Products/Debug-iphonesimulator/TempContacts.app && xcrun simctl launch --wait-for-debugger booted com.mp.TempContacts",
      "group": {
        "kind": "build",
        "isDefault": true
      }
    }
```
- `simctl install booted build/Build/Products/Debug-iphonesimulator/TempContacts.app` installs on simulator executable from provided path build in previous steps
- `xcrun simctl launch booted com.mp.TempContacts` runs app based on given app bundle id (as in `project.yaml`)
- `--wait-for-debugger` argument hold app execution until debugger is attached


Example `tasks.json` could look like that: 
```json
{
  // See https://go.microsoft.com/fwlink/?LinkId=733558
  // for the documentation about the tasks.json format
  "version": "2.0.0",
  "tasks": [
    { // Building an app 
      "label": "Build ios",
      "type": "shell",
      "command": "xcodebuild -scheme 'TempContacts' -derivedDataPath build -configuration Debug -project TempContacts.xcodeproj -destination 'platform=iOS Simulator,name=iPhone 11' build",
      "dependsOn": [
        "Gen XCodeproj"
      ],
      "group": {
        "kind": "build",
        "isDefault": true
      }
    },
    { // generating xcode project
      "label": "Gen XCodeproj",
      "type": "shell",
      "command": "xcodegen generate --use-cache"
    },
    { // running simulator
      "label": "runSimulator",
      "type": "shell",
      "command": "open /Applications/Xcode.app/Contents/Developer/Applications/Simulator.app/"
    },
    { // Running an app
      "label": "Run ios",
      "type": "shell",
      "dependsOn": [
        "Build ios",
        "runSimulator"
      ],
      "command": "xcrun simctl install booted build/Build/Products/Debug-iphonesimulator/TempContacts.app && xcrun simctl launch booted com.mp.TempContacts",
      "group": {
        "kind": "build",
        "isDefault": true
      },
      "problemMatcher": []
    },
    { // Launch app and wait for debugger
      "label": "Debug ios",
      "type": "shell",
      "dependsOn": [
        "Build ios",
        "runSimulator"
      ],
      "command": "xcrun simctl install booted build/Build/Products/Debug-iphonesimulator/TempContacts.app && xcrun simctl launch --wait-for-debugger booted com.mp.TempContacts",
      "group": {
        "kind": "build",
        "isDefault": true
      }
    }
  ]
}
```

Now you can build or buil & run in simulator just by pressing `Cmd+Shift+B` and selecting either `Run ios` or `Build ios` from task list.

## Configuring LLDB for debuging 

In order to enable debugging for executables you we need to set proper LLDB library for debugger plugin, add this line in `settings.json` pointing to LLDB library in your toolchain : 

`"lldb.library": "/Applications/Xcode.app/Contents/SharedFrameworks/LLDB.framework/Versions/A/LLDB", `

Next, add this vscode launch configuration: 

```json
{
  // Use IntelliSense to learn about possible attributes.
  // Hover to view descriptions of existing attributes.
  // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
  "version": "0.2.0",
  "configurations": [
    {
      "type": "lldb",
      "request": "attach",
      "name": "Launch",
      "program": "TempContacts.app/TempContacts", //Program exeutable process name
      "waitFor": true,
      "preLaunchTask": "Debug ios"
    }
  ]
}
```
`"program": "TempContacts.app/TempContacts"` - your process name would usually look like `PRODUCT_NAME.app/PRODUCT_NAME`

Now, launching this task will build and run ios app waiting for debugger to attach, debugger 
will attach to process based on app name.

## What works, what not, troubleshooting and tips

Sourcekit-LSP heavily relies on index database build with swift compilation, it's trigger when initializing server. So far I noticed that adding a new file requires reloading VSCode window for server to properly repopulate workspace sources. Also, if some semanitc tokens are missing, or find references shows nothing, rebulding with `swift build` command helps with completing missing indexes.

## 