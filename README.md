
# Wololo iOS

  

This document explains the operation of the iOS pipeline within Wololo, it covers from the moment Unity produces an Xcode Project until the IPA file is uploaded to TestFlight.

  

It is necessary to clarify that although the pipeline is integrated with **Azure Pipeline**, the code is completely agnostic to the platform, the dependencies it has with said platform are completely reproducible in any environment.

### Main Pipeline File

  

Although the pipeline is agnostic to the platform, it is worth referencing the [YAML](https://github.com/fortunacio/wololo-azure-pipeline/blob/master/templates/main-pipeline.yml) that we use as the main pipeline within **Azure Pipeline**, it uses open source [Tasks](https://github.com/microsoft/azure-pipelines-tasks), some of them were modified by us, and it is worth mentioning the changes that we did and why they were made.

  

### Install Certificates on Mac VM ( .P12 and .mobileprovisioning)

  

We used two well-documented azure tasks with no modifications:

  

-  [P12 certificate](https://github.com/microsoft/azure-pipelines-tasks/tree/master/Tasks/InstallAppleCertificateV2)

-  [ProvisioningProfile](https://github.com/microsoft/azure-pipelines-tasks/tree/master/Tasks/InstallAppleProvisioningProfileV1)

  

### Cocoapods dependencies

  

It is very common in unity projects that [Unity Resolver](https://github.com/googlesamples/unity-jar-resolver
) is used to handle all the dependencies of a project, it integrates Cocoapod to be able to deal with the problem. It is also common that developers themselves choose to use cocoapods to better manage their dependencies, for this reason we support it in our pipeline as follows:

  

- Run [`prepare-workspace.sh`](https://github.com/fortunacio/wololo-azure-pipeline/blob/master/scripts/prepare-workspace.sh) to check if `Podfile` exist and inject the new [Cocoapod CDN](https://cdn.cocoapods.org/) to avoid download the [Cocoapods Master Repo](https://github.com/CocoaPods/Specs).

- Check if some Cocoapod cache exist (`Podfile.lock` is used as cache key) using [Cache Task](https://github.com/microsoft/azure-pipelines-tasks/tree/master/Tasks/CacheV2)

- Run `pod install`

### Code Signing & Build

  We use a modified version of the [Xcode Task](https://github.com/fortunacio/xcode-task-wololo-azure) of azure, because in versions of unity >= 2019.3 the process to sign the project changes, according to the [official documentation](https://docs.unity3d.com/2019.3/Documentation/Manual/StructureOfXcodeProject.html) these are the required changes:

> When you use command line arguments to specify build settings, these affect all Xcode project targets. To prevent this, some build settings have suffixed versions which you can use to specify which target your build settings affect. This is implemented through User-Defined Settings (*APP suffix used for application target and *FRAMEWORK suffix for framework target).

  

>When building with xcodebuild, use suffixed versions for:

***PRODUCT_NAME -> PRODUCT_NAME_APP***

***PROVISIONING_PROFILE -> PROVISIONING_PROFILE_APP***

***PROVISIONING_PROFILE_SPECIFIER -> PROVISIONING_PROFILE_SPECIFIER_APP***

***OTHER_LDFLAGS -> OTHER_LDFLAGS_FRAMEWORK***

  

The code of this task is well commented and is quite simple, it use the `xcodebuild` command with the following parameters:
```
/usr/bin/xcodebuild 
-sdk iphoneos 
-configuration Release 
-workspace Unity-iPhone.xcworkspace 
-scheme Unity-iPhone 
build 
CODE_SIGN_STYLE=Manual 
CODE_SIGN_IDENTITY=<signing identity override with which to sign the build>
PROVISIONING_PROFILE_APP=<prifile uuid installed on VM> 
PROVISIONING_PROFILE_SPECIFIER_APP= (let it empty to avoid a well-known error.)
```
#### The incremental build problem, Ccache to the rescue :fire:

In the fight to obtain the optimal build time for our users, a solution came to mind, the [intemental build of unity](https://docs.unity3d.com/ScriptReference/PlayerSettings.SetIncrementalIl2CppBuild.html) but we quickly discarded that idea, since we maintain a hybrid ecosystem, where the xcode project is generated in a Linux machine and is later transferred to a Mac machine, **the problem?** the incremental build of unity is not supported by Linux, **the solution?** our great friend [Ccache](https://ccache.dev/).
The way we implement Ccache is really simple, with this solution we got more than 80% cache hit and decreased our build time by 70%,  the following steps do the magic:

 - Before starting the build process, we inject the [ccache-clang](https://github.com/fortunacio/wololo-azure-pipeline/blob/master/scripts/ccache-clang) script to the project to configure ccache:
 ````
 #!/bin/sh
if type -p ccache >/dev/null 2>&1; then
	export CCACHE_MAXSIZE=20G
	export CCACHE_CPP2=true
	export CCACHE_COMPILERCHECK=content
	export CCACHE_SLOPPINESS=file_macro,locale,time_macros,include_file_mtime,include_file_ctime
	export CCACHE_NOHASHDIR
	exec ccache /usr/bin/clang "$@"
else
	exec clang "$@"
fi
````
 - We define one more argument in xcodebuild to tell it where the script is located: CC=ccache-clang.
 - Start the build process.
 
As we can see, ccache does not imply greater complexity, the key is to find the best configuration for unity projects.

### Upload IPA file to TestFlight

We used the well-documented [App Store Release Task](https://github.com/microsoft/app-store-vsts-extension/tree/master/Tasks/app-store-release) with no modifications. The builds are uploaded to the Alpha channel.
The type of authorization we prefer for this task is **apple store username and password**, we require users to create a new account with sufficient permissions to be able to push to TestFlight.

#### Apple Two-factor auth, the most powerful headache :anger:.
New apple accounts come with Two-factor authentication enabled by default, and it is not possible to disable it, or in case you really don't want to disable it for personal reasons, this becomes a headache.

Apple does not provide any type of ready-to-go api to deal with this problem, therefore the best alternative we had was to use [spaceship](https://github.com/fastlane/fastlane/tree/master/spaceship), a tool integrated in fastlane. This tool is written in Ruby and we decided to translate it to Python and integrate it with our backend to support Two-factor auth with a [single endpoint](#endpoint-token) and in a simple way.

The problem with this solution is that Apple gives you a Token that expires after 30 days, therefore we also implement an [endpoint](#endpoint-refresh-token) that involves part of the Spaceship code with our modifications to avoid the user having to go through the entire Two-factor process.