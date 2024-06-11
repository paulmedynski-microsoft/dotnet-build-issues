# dotnet-build-issues

A project to help reproduce issues with the dotnet build command.

## No-Restore & Artifacts-Path Incompatible

https://github.com/dotnet/sdk/issues/41530

The ```dotnet restore``` and ```dotnet build``` commands don't play nicely when
you're trying to use lock files and an artifacts path.

```
$ dotnet --version
8.0.300
```

Using lock files for ```dotnet restore``` is recommended for reproducible
builds, and ensures that all NuGet packages are identical to those used for
previous builds.  You can generate a lock file using the ```--use-lock-file```
argument:

```
$ dotnet restore --use-lock-file
  Determining projects to restore...
  Restored /home/paul/dev/paulmedynski/dotnet-build-issues/dotnet-build-issues.csproj (in 224 ms).

$ cat packages.lock.json
{
  "version": 1,
  "dependencies": {
    "net8.0": {
      "NuGet.Common": {
        "type": "Direct",
        "requested": "[6.10.0, )",
        "resolved": "6.10.0",
        "contentHash": "ujuzfDwUylDALwZmqRi7I5Jx4E9/8vR2c0Hq8zRj8zCkR8KarSp84WPJ2uE/qwAOjpBYGek06wWgGJ3ABQ/WxA==",
        "dependencies": {
          "NuGet.Frameworks": "6.10.0"
        }
      },
      "NuGet.Frameworks": {
        "type": "Transitive",
        "resolved": "6.10.0",
        "contentHash": "bKaC87Q8rxK7ozFN9Eyo0YVUmd4r2s8pbNxHI7sHRqL16OAP0yEdCU5wpDkG9cKLHpZCahgoQUfAVC0UadVU+A=="
      }
    }
  }
```

When performing a build, you can ensure that those exact NuGet packages are
restored by using the ```--locked-mode``` argument:

```
$ dotnet restore --locked-mode
  Determining projects to restore...
  Restored /home/paul/dev/paulmedynski/dotnet-build-issues/dotnet-build-issues.csproj (in 219 ms).
```

Restoring places some  meta-data files (like ```project.assets.json```) in
```obj/```.

```
$ ls -al obj/
total 44
drwxr-xr-x 3 paul paul 4096 Jun 11 14:16 .
drwxr-xr-x 5 paul paul 4096 Jun 11 14:14 ..
drwxr-xr-x 3 paul paul 4096 Jun 11 13:54 Debug
-rw-r--r-- 1 paul paul 2176 Jun 11 14:16 dotnet-build-issues.csproj.nuget.dgspec.json
-rw-r--r-- 1 paul paul 1110 Jun 11 14:14 dotnet-build-issues.csproj.nuget.g.props
-rw-r--r-- 1 paul paul  149 Jun 11 14:14 dotnet-build-issues.csproj.nuget.g.targets
-rw-r--r-- 1 paul paul 3621 Jun 11 14:16 project.assets.json
-rw-r--r-- 1 paul paul  396 Jun 11 14:16 project.nuget.cache
```

When building, we must specify ```--no-restore```, since ```dotnet build```
doesn't support ```--locked-mode``` and it would otherwise perform an implicit
restore, potentially clobbering the carefully locked packages.

```
$ dotnet build -c Release --no-restore
  dotnet-build-issues -> /home/paul/dev/paulmedynski/dotnet-build-issues/bin/Debug/net8.0/dotnet-build-issues.dll

Build succeeded.
    0 Warning(s)
    0 Error(s)

Time Elapsed 00:00:02.27
```

.NET 8 added the ```--artifacts-path``` option to the ```dotnet build``` and
```dotnet publish``` commands.  This places all build and publish output into
sub-directories under the given path.

However, when using ```--artifacts-path```, the build expects the restore
meta-data files to exist _within_ that artifacts path, but ```dotnet restore```
puts them in ```obj/```.

```
$ dotnet build -c Release --no-restore --artifacts-path artifacts
/usr/share/dotnet/sdk/8.0.300/Sdks/Microsoft.NET.Sdk/targets/Microsoft.PackageDependencyResolution.targets(266,5): error NETSDK1004: Assets file '/home/paul/dev/paulmedynski/dotnet-build-issues/artifacts/obj/dotnet-build-issues/project.assets.json' not found. Run a NuGet package restore to generate this file. [/home/paul/dev/paulmedynski/dotnet-build-issues/dotnet-build-issues.csproj]

Build FAILED.
```

It would seem that ```--no-restore``` and ```--artifacts-path``` are
incompatible for ```dotnet build```.

I can think of 2 solutions:

1.  Add ```--artifacts-path``` to ```dotnet restore``` and have it place the required meta-data files where ```dotnet build``` expects to find them.

2.  Add ```--locked-mode``` to ```dotnet build``` to instruct implicit restores to obey the lock files.

Either of these options would permit lock files and artifacts paths to co-exist.
I think the first option is the most flexible since it allows you to restore
separately from building.

A workaround for a simple single-project solution like this is to use the ```-p ArtifactsPath``` option.

```
$ dotnet restore --locked-mode -p ArtifactsPath=artifacts
$ dotnet build -c Release --no-restore --artifacts-path artifacts
  dotnet-build-issues -> /home/paul/dev/paulmedynski/dotnet-build-issues/artifacts/bin/dotnet-build-issues/debug/dotnet-build-issues.dll

Build succeeded.
    0 Warning(s)
    0 Error(s)

Time Elapsed 00:00:01.03
```
