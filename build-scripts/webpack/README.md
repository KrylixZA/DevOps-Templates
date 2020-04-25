# Webpack
This directory contains the necessary scripts to setup CI for a UI project to build and publish packages to the organization nuget feed.

# Steps
1. Create a `tools` directory at the root-level of your repository, and paste the [nuget.exe](nuget.exe) file in the tools directory of your repository.
2. Create a `build` directory at the root-level of your repository. In this directory you will create the desired `.nuspec` files that you need to pack your project.
3. Copy and paste the following files into the root of your repository:
    * [azure-pipelines.yml](azure-pipelines.yml)
    * [build-bootstrapper.ps1](build-bootstrapper.ps1)
    * [build.ps1](build.ps1)
    * [unit-test.ps1](unit-test.ps1) (if you intend to write unit tests for your project)
    * [version.ps1](version.ps1)

You can now test your build locally by running the [build-bootstrapper.ps1](build-bootstrapper.ps1) with the following PowerShell:
``` PowerShell
.\build-bootstrapper.ps1 -Actions "build" # To only test the build
.\build-bootstrapper.ps1 -Actions "unit-test" # To only test the unit testing
.\build-bootstrapper.ps1 # To test build & unit test
```