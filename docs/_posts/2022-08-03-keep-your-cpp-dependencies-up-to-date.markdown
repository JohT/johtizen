---
layout: post
title:  "Keep your C++ dependencies up-to-date with Renovate & CPM"
date:   2022-08-03 08:30:00 +0100
categories: automation
tags: renovate cpm dependency version
author: JohT
comments-issue-id: 25
---

Looking for new versions of libraries and updating them can be a tedious task, especially if you have to deal with multiple dependencies and many repositories.
[Renovate][Renovate] is a tool that can scan your dependencies and update them to the latest version. But can this also be utilized in a C++ project with [CPM.cmake][CPM.cmake]?

### Table of Contents
{:.no_toc}
1. A markdown unordered list which will be replaced with the table of contents.
{:toc}

{% raw %}

## Prerequisites

- C++ project with [CMake [3]][CMake] as build tool.

## External Dependencies

### What is [CPM.cmake [2]][CPM.cmake]?

cpm is ...
> CMake's missing package manager. A small CMake script for setup-free, cross-platform, reproducible dependency management.

A key difference to other package managers is:
> Any downloadable project or resource can be added as a version-controlled dependency though CPM.

### Why not just use [Git Submodules [11]][GitSubmodules]?

A common practice to include external sources in C++ project is to use [Git Submodules][GitSubmodules]. This is a good solution for many projects:
- It doesn't introduce any additional dependencies
- Continuous Integration is easy to setup.
- The build is reproducible since submodules are tied to specific Git commits.

But there are also some drawbacks:
- It isn't possible to use Git version tags directly (only commit SHA's).
- It can't be used to download already built packages.
- It can't be used for sources that are not available in Git.

In the end it is a choice of what suits the project best. [Automated Dependency Updates for Git Submodules [12]][RenovateGitSubmodules] goes further into details on how to setup automatic update for Git Submodules. Otherwise continue reading if you are interested in what [CPM.cmake][CPM.cmake] can do.

### How to use [CPM.cmake][CPM.cmake]?

These few lines in [CMake][CMake]'s `CMakeLists.txt` are all you need to setup [CPM.cmake][CPM.cmake]:

```cmake
set(CPM_DOWNLOAD_VERSION 0.27.2) 
set(CPM_DOWNLOAD_LOCATION "${CMAKE_BINARY_DIR}/cmake/CPM_${CPM_DOWNLOAD_VERSION}.cmake")

if(NOT (EXISTS ${CPM_DOWNLOAD_LOCATION}))
    message(STATUS "Downloading CPM.cmake")
    file(DOWNLOAD https://github.com/TheLartians/CPM.cmake/releases/download/v${CPM_DOWNLOAD_VERSION}/CPM.cmake ${CPM_DOWNLOAD_LOCATION})
endif()

include(${CPM_DOWNLOAD_LOCATION})
```

As an example, adding Catch2 for unit testing looks like this (shorthand syntax):

```cmake
CPMAddPackage("gh:catchorg/Catch2@3.1.0")
```

For more information have a look at this article: [CPM: An Awesome Dependency Manager for C++ with CMake [13]][CPMAwesome]

### Why [CPM.cmake][CPM.cmake]?

[CPM.cmake][CPM.cmake] can be used like [Git Submodules][GitSubmodules] to include sources from other Git Repositories. It can furthermore be used to download already built packages and download sources that are not available in Git. It can work with git tags and commit references. [CPM.cmake][CPM.cmake] wraps [CMake's FetchContent [14]][CMakeFetchContent] to simplify downloading external dependencies. It doesn't depend on a centralized package repository nor does it require any metadata to support [CPM.cmake][CPM.cmake].

### [CPM Package Lock [15]][CPMPackageLock]

[CPM.cmake][CPM.cmake] provides a way to list all dependencies in one `package-lock.cmake` file. These can then easily be referenced in `CMakeLists.txt` by their name where needed. This mimics other package managers and has several advantages. It is easier to read, simplifies the process of updating dependencies and integrates much better with other tools.

- Add `CPMUsePackageLock` right after including [CPM.cmake][CPM.cmake]:
```cmake
include(cmake/CPM.cmake)
CPMUsePackageLock(package-lock.cmake)
```

- Build the utility target:
```shell
cmake -H. -Bbuild
cmake --build build --target cpm-update-package-lock
```

The resulting `package-lock.cmake` may then look for example like this:
```cmake
# CPM Package Lock
# This file should be committed to version control

# Catch2
CPMDeclarePackage(Catch2
  VERSION 3.1.0
  GITHUB_REPOSITORY catchorg/Catch2
  EXCLUDE_FROM_ALL YES
)
```

The library is then referenced by name within `CMakeLists.txt` like this:
```cmake
CPMGetPackage(Catch2)
```

## Version Updates

It is a tedious and time consuming task to update dependencies to their latest version manually. Tools like [Renovate][Renovate] automate this process by scanning your dependencies and updating them to the latest version. Staying up-to-date is essential when it comes to security updates and bug fixes. It is also easier to continuously update in small steps. Last but not least an update might come in handy when there are useful new features.

### What is [Renovate [1]][Renovate]?

Renovate is a ...

> universal dependency update tool that fits into your workflows

... and let you ...
> get automated Pull Requests to update your dependencies

### How to use [Renovate][Renovate]?

If your repository is on [GitHub [6]][GitHub], [Renovate GitHub App [4]][RenovateGitHubApp] is free to install. [Installing and onboarding Renovate into repositories [5]][RenovateGitHubAppInstall] gives you more information about that.
 
The configuration for Renovate is usually found in the file `renovate.json`. 
For further options see [Renovate Configuration Options [7]][RenovateConfiguration]. The most basic configuration looks like this: 

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:base"
  ]
}
```

## How to use [Renovate][Renovate] with [CPM.cmake][CPM.cmake]?

[CPM.cmake [2]][CPM.cmake] isn't supported by [Renovate][Renovate] yet (August 2022). Fortunately, there is a workaround for that. As described in [Custom Manager Support using Regex [8]][RenovateCustomManager], [Regular Expressions [10]][RegularExpression] can be used to identify dependencies and then find and replace their versions.

<span style="font-size:1.4em;">&#9432;</span> 
Note that it is recommended to use a tool like [regex101.com](https://regex101.com) to get immediate feedback when developing [Regular Expressions][RegularExpression].

### [Renovate Regex Manager][RenovateCustomManager] for [CPM Package Lock][CPMPackageLock] VERSION

The following `package-lock.cmake` entry shows a dependency to a [GitHub][GitHub] repository with a specific version:

```cmake
CPMDeclarePackage(Catch2
  VERSION 3.1.0
  GITHUB_REPOSITORY catchorg/Catch2
  EXCLUDE_FROM_ALL YES
)
```

Use this `renovate.json` to update the dependency's version with a pull request whenever there is a new release:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:base",
  ],
  "regexManagers": [
    {
      "fileMatch": ["(^|/)package-lock\\.cmake$"],
      "matchStrings": [
        "CPMDeclarePackage\\s*\\(\\s*\\w+\\s*.*VERSION\\s+\"?(?<currentValue>.*?)\"?\\s+GITHUB_REPOSITORY\\s+\"?(?<depName>.*?)\"?[\\s\\)]",
        "CPMDeclarePackage\\s*\\(\\s*\\w+\\s*.*GITHUB_REPOSITORY\\s+\"?(?<depName>.*?)\"?\\s+VERSION\\s+\"?(?<currentValue>.*?)\"?[\\s\\)]"
      ],
      "datasourceTemplate": "github-releases",
      "extractVersionTemplate": "^v?(?<version>.*?)$"
    }
  ]
}
```

<span style="font-size:1.4em;">&#9432;</span> 
Note that there are two similar [Regular Expressions][RegularExpression] to be independent from the order of `VERSION` and `GITHUB_REPOSITORY`.

<span style="font-size:1.4em;">&#9432;</span> 
Note that [CMake][CMake] values can be given with or without surrounding double quotes. This is reflected within the [Regular Expression][RegularExpression] by `\"?`.

<span style="font-size:1.4em;">&#9432;</span> 
Note that `extractVersionTemplate` is necessary to extract the version including the "v" prefix from the GitHub release tag whereas it is given without the prefix.

<span style="font-size:1.4em;">&#9432;</span> 
Note that it is also possible to observe all tags and not only those published as a release using `"datasourceTemplate": "github-tags"`.
 
### [Renovate Regex Manager][RenovateCustomManager] for [CPM Package Lock][CPMPackageLock] GIT_TAG

The following `package-lock.cmake` entry shows a dependency to a [GitHub][GitHub] repository with a specific git tag (without the "v" prefix):

```cmake
CPMDeclarePackage(JUCE
  GIT_TAG 7.0.1
  GITHUB_REPOSITORY juce-framework/JUCE
  EXCLUDE_FROM_ALL YES
)
```

Use this regexManager in `renovate.json` to update the dependency's git tag with a pull request whenever there is a new release:

```json
{
  "fileMatch": ["(^|/)package-lock\\.cmake$"],
  "matchStrings": [
    "CPMDeclarePackage\\s*\\(\\s*\\w+\\s*.*GIT_TAG\\s+\"?(?<currentValue>.*?[^0-9a-f\\s]+.*|.{1,4}?)\"?\\s+GITHUB_REPOSITORY\\s+\"?(?<depName>.*?)\"?\\s+",
    "CPMDeclarePackage\\s*\\(\\s*\\w+\\s*.*GITHUB_REPOSITORY\\s+\"?(?<depName>.*?[^0-9a-f\\s]+.*|.{1,4}?)\"?\\s+GIT_TAG\\s+\"?(?<currentValue>.*?)\"?\\s+"
  ],
  "datasourceTemplate": "github-releases"
}
```

<span style="font-size:1.4em;">&#9432;</span> 
Note that the GIT_TAG value needs to contain at least one character that distinguishes it from a Git SHA. Also values with less than 4 characters are considered Non-SHA. This is reflected within the [Regular Expression][RegularExpression] by `.*?[^0-9a-f\\s]+.*|.{1,4}`. GIT_TAG supports both tags and commits. They need to be distinguished because they need to be updated differently.
 
### [Renovate Regex Manager][RenovateCustomManager] for [CPM Package Lock][CPMPackageLock] GIT_TAG commit SHA

The following `package-lock.cmake` entry shows a dependency to a [GitHub][GitHub] repository with a specific git commit:

```cmake
CPMDeclarePackage(span
 GIT_TAG 836dc6a0efd9849cb194e88e4aa2387436bb079b
 GITHUB_REPOSITORY tcbrindle/span
 EXCLUDE_FROM_ALL YES
)
```

Use this regexManager in `renovate.json` to update the dependency's git SHA with a pull request whenever there is a new commit in the "master" branch:

```json
{
  "fileMatch": ["(^|/)package-lock\\.cmake$"],
  "matchStrings": [
    "CPMDeclarePackage\\s*\\(\\s*\\w+\\s*.*GIT_TAG\\s+\"?(?<currentDigest>[0-9a-f]{5,40}?)\"?\\s+GITHUB_REPOSITORY\\s+?\"?(?<depName>.*?)\"?\\s+",
    "CPMDeclarePackage\\s*\\(\\s*\\w+\\s*.*GITHUB_REPOSITORY\\s+\"?(?<depName>[0-9a-f]{5,40}?)\"?\\s+GIT_TAG\\s+?\"?(?<currentDigest>.*?)\"?\\s+"
  ],
  "datasourceTemplate": "git-refs",
  "depNameTemplate": "https://github.com/{{{depName}}}.git",
  "currentValueTemplate": "master"
}
```

<span style="font-size:1.4em;">&#9432;</span> 
Note that the GIT_TAG value needs to be a Git SHA. This is reflected within the [Regular Expression][RegularExpression] by `[0-9a-f]{5,40}`. GIT_TAG supports both tags and commits. They need to be distinguished because they need to be updated differently.

<span style="font-size:1.4em;">&#9432;</span> 
Note that `currentDigest` is used instead of `currentValue` to reflect the fact that it is a unique SHA and not a version (that could contain e.g. wildcards). `currentValue` won't work here.

<span style="font-size:1.4em;">&#9432;</span> 
Note that `depNameTemplate` is used to construct the dependency's Git URL since [Renovate][Renovate] doesn't support a dataSource like "github-refs" yet. Nevertheless this can be very useful when a custom Git URL is needed.

<span style="font-size:1.4em;">&#9432;</span> 
Note that `currentValueTemplate` needs to be changed to "main" if this is the branch you want to observe.

### [Renovate Regex Manager][RenovateCustomManager] for [GitHub][GitHub] download links in [CMake][CMake]

Before it can be used, [CPM.cmake][CPM.cmake] needs to be downloaded. This can be done at build time with [CMake][CMake]. The following `CMakeLists.txt` line shows a simple uncached solution to download [CPM.cmake][CPM.cmake] from a specific [GitHub][GitHub] release. A [cached solution with variable](#renovate-regex-manager-for-github-downloads-with-a-version-variable-in-cmake) is shown further below.

```cmake
file(DOWNLOAD https://github.com/cpm-cmake/CPM.cmake/releases/download/v0.35.4/CPM.cmake ${CMAKE_BINARY_DIR}/cmake/CPM.cmake)
```

Use this regexManager in `renovate.json` to update [CPM.cmake][CPM.cmake] with a pull request whenever there is a new release:

```json
{
  "fileMatch": ["(^|/)\\w+\\.cmake$", "(^|/)CMakeLists\\.cmake$"],
  "matchStrings": [
    "https:\\/\\/github\\.com\\/(?<depName>[^{}]*?)\\/releases\\/download\\/(?<currentValue>[^{}]*?)\\/"
  ],
  "datasourceTemplate": "github-releases"
}
```

<span style="font-size:1.4em;">&#9432;</span> 
Note that this [Regular Expression][RegularExpression] can be used to update any [GitHub][GitHub] release download link, not just [CPM.cmake][CPM.cmake].

<span style="font-size:1.4em;">&#9432;</span> 
Note that this doesn't work when the version is taken from a [CMake][CMake] variable. This is assured within the [Regular Expression][RegularExpression] with `[^{}]*`.

### [Renovate Regex Manager][RenovateCustomManager] for [GitHub][GitHub] downloads with a version variable in [CMake][CMake] 

To download [CPM.cmake][CPM.cmake] with the version specified in a [CMake][CMake] variable as shown above in [How to use CPM.cmake?](#how-to-use-cpmcmake), there need to be some sort of convention to be able to update it with [Renovate](Renovate). 

The following regexManager in `renovate.json` shows one possible solution to update the version variable for the download with a pull request whenever there is a new release:

```json
{
  "fileMatch": ["(^|/)\\w+\\.cmake$", "(^|/)CMakeLists\\.cmake$"],
  "matchStrings": [
    "set\\s*\\(\\s*\\w+VERSION\\s+\"?(?<currentValue>[^v$][^${}]*?)\"?\\s*\\)[\\S\\s]*https:\\/\\/github.com\\/(?<depName>.*?)\\/releases\\/download\\/v\\$\\{\\w+VERSION\\}"
  ],
  "datasourceTemplate": "github-releases",
  "extractVersionTemplate": "^v?(?<version>.*?)$"
}
```

<span style="font-size:1.4em;">&#9432;</span> 
Note that the variable containing the version needs to have a name ending with "VERSION". It also needs to be defined right before the download. 

<span style="font-size:1.4em;">&#9432;</span> 
Note that since [Regular Expression RE2][RE2] doesn't support backreferences, the name of the version variable is only checked to end with "VERSION", it isn't checked to be actual equal to its definition.

### Real world example

The repository [speclet](https://github.com/JohT/speclet) shows a real world example
with all the above mentioned snippets in these files:

- [renovate.json](https://github.com/JohT/speclet/blob/main/renovate.json)
- [package-lock.cmake](https://github.com/JohT/speclet/blob/main/package-lock.cmake)
- [Environment.cmake](https://github.com/JohT/speclet/blob/main/cmake/Environment.cmake)
 
## Summary

To answer the question of the introduction: C++ projects can of course also benefit from a modern tool like [Renovate][Renovate]. Even if package managers like [CPM.cmake][CPM.cmake] aren't supported yet, automatic version updates can still be customized using [Renovate Regex Managers][RenovateCustomManager].

[renovate.json](https://github.com/JohT/speclet/blob/main/renovate.json) shows a real world example with the whole configuration. It relies on [package-lock.cmake](https://github.com/JohT/speclet/blob/main/package-lock.cmake) containing the description of all dependencies. 

It should also be possible to extend the configuration for projects that don't use [CPM Package Lock][CPMPackageLock] as well as for those that use [CPM.cmake's][CPM.cmake] shorthand syntax in future.

<br>
 
{% endraw %}

----
## References
- [[1] Renovate][Renovate]  
https://github.com/renovatebot/renovate
- [[2] CPM.cmake][CPM.cmake]  
https://github.com/cpm-cmake/CPM.cmake
- [[3] CMake][CMake]  
https://cmake.org
- [[4] Renovate GitHub App][RenovateGitHubApp]  
https://github.com/apps/renovate
- [[5] Installing and onboarding Renovate into repositories][RenovateGitHubAppInstall]   
https://docs.renovatebot.com/getting-started/installing-onboarding
- [[6] GitHub][GitHub]  
https://github.com
- [[7] Renovate Configuration Options][RenovateConfiguration]  
https://docs.renovatebot.com/configuration-options
- [[8] Renovate Custom Manager Support using Regex][RenovateCustomManager]   
https://docs.renovatebot.com/modules/manager/regex
- [[10] Regular Expression][RegularExpression]   
https://en.wikipedia.org/wiki/Regular_expression
- [[11] GitSubmodules][GitSubmodules]   
https://git-scm.com/book/en/v2/Git-Tools-Submodules
- [[12] Automated Dependency Updates for Git Submodules][RenovateGitSubmodules]   
https://docs.renovatebot.com/modules/manager/git-submodules
- [[13] CPM: An Awesome Dependency Manager for C++ with CMake][CPMAwesome]   
https://medium.com/swlh/cpm-an-awesome-dependency-manager-for-c-with-cmake-3c53f4376766
- [[14] CMake FetchContent][CMakeFetchContent]   
https://cmake.org/cmake/help/latest/module/FetchContent.html
- [[15] CPM Package Lock][CPMPackageLock]   
https://github.com/cpm-cmake/CPM.cmake/wiki/Package-lock
- [[16] Regular Expression RE2][RE2]   
https://github.com/google/re2/wiki/WhyRE2

[CPM.cmake]: https://github.com/cpm-cmake/CPM.cmake
[CPMAwesome]: https://medium.com/swlh/cpm-an-awesome-dependency-manager-for-c-with-cmake-3c53f4376766
[CPMPackageLock]: https://github.com/cpm-cmake/CPM.cmake/wiki/Package-lock
[CMake]: https://cmake.org
[CMakeFetchContent]: https://cmake.org/cmake/help/latest/module/FetchContent.html
[GitHub]: https://github.com
[GitSubmodules]: https://git-scm.com/book/en/v2/Git-Tools-Submodules
[RegularExpression]: https://en.wikipedia.org/wiki/Regular_expression
[Renovate]: https://github.com/renovatebot/renovate
[RenovateConfiguration]: https://docs.renovatebot.com/configuration-options
[RenovateCustomManager]: https://docs.renovatebot.com/modules/manager/regex
[RenovateGitHubApp]: https://github.com/apps/renovate
[RenovateGitHubAppInstall]: https://docs.renovatebot.com/getting-started/installing-onboarding
[RenovateGitSubmodules]: https://docs.renovatebot.com/modules/manager/git-submodules
[RE2]: https://github.com/google/re2/wiki/WhyRE2