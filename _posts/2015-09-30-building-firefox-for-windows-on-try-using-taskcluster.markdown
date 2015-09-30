---
layout: post
title:  "Building Firefox for Windows™ on Try using TaskCluster"
date:   2015-09-30 14:08:53
categories: general taskcluster
---

![Firefox on Windows screenshot](/assets/2014-06-20-10-38-11-106a6c.png)

Try them out for yourself!
--------------------------

Here are the try builds we have created. They were built from the official in-tree [mozconfigs](https://hg.mozilla.org/mozilla-central/file/default/browser/config/mozconfigs) that we use for the builds running in Buildbot.

* [win32 opt](https://queue.taskcluster.net/v1/task/ARNCa6tfSDuXMxkzxBjUTQ/runs/0/artifacts/public/build/firefox-43.0a1.en-US.win32.zip)
* [win32 debug](https://queue.taskcluster.net/v1/task/ZbyHXDfCTeG40Khq8sDI9w/runs/0/artifacts/public/build/firefox-43.0a1.en-US.win32.zip)
* [win64 opt](ihttps://queue.taskcluster.net/v1/task/YlS2Tx5cSTioK144E1TnOA/runs/0/artifacts/public/build/firefox-43.0a1.en-US.win64.zip)
* [win64 debug](https://queue.taskcluster.net/v1/task/Ar_6p6QCQ02_xYIaMc_Dlw/runs/0/artifacts/public/build/firefox-43.0a1.en-US.win64.zip)

Set up your own Windows™ Try tasks
----------------------------------

We are porting over all of Mozilla's CI tasks to [TaskCluster](https://docs.taskcluster.net), including Windows™ builds and tests.

Currently Windows™ and OS X tasks still run on our legacy Buildbot infrastructure. This is about to change.

In this post, I am going to talk you through how I set up Firefox Desktop builds in TaskCluster on [Try](https://wiki.mozilla.org/ReleaseEngineering/TryServer#Try_Server). In future, the TaskCluster builds should replace the existing Buildbot builds, even for releases. Getting them running on Try was the first in a long line of many steps.

Spoiler alert: [https://treeherder.mozilla.org/#/jobs?repo=try&revision=fc4b30cc56fb](https://treeherder.mozilla.org/#/jobs?repo=try&revision=fc4b30cc56fb)

Using the right Worker
----------------------

In TaskCluster, Linux tasks run in a [docker](https://www.docker.com/) container. This doesn't work on Windows, so we needed a different strategy.

TaskCluster defines the role of a [_Worker_](http://docs.taskcluster.net/queue/worker-interaction/) as component that is able to claim tasks from the Queue, execute them, publish artifacts, and report back status to the Queue.

For Linux, we have the [Docker Worker](http://docs.taskcluster.net/workers/docker-worker/). This is the component that takes care of executing Linux tasks inside a docker container. Since everything takes place in a container, consecutive tasks cannot interfere with each other, and you are guaranteed a clean environment.

This year I have been working on the [Generic Worker](http://taskcluster.github.io/generic-worker/). This takes care of running TaskCluster tasks on other platforms.

For Windows, we have a different isolation strategy: since we cannot yet easily run inside a container, the Generic Worker will create a new Windows user for each task it runs.

This user will have its own home directory, and will not have privileged access to the host OS. This means, it should not be able to make any persistent changes to the host OS that will outlive the lifetime of the task. The user only is able to affect `HKEY_CURRENT_USER` registry settings, and write to its home folder, which are both purged after task completion.

In other words, although not running in a container, the Generic Worker offers isolation to TaskCluster tasks by virtue of running each task as a different, custom created OS user with limited privileges.

Creating a Worker Type
----------------------

TaskCluster considers a Worker Type as an entity which belongs to a [Provisioner](http://docs.taskcluster.net/aws-provisioner/api-docs/), and represents a host environment and hardware context for running one or more Workers. This is the Worker Type that I set up:

{% highlight json %}
{
  "workerType": "win2012r2",
  "minCapacity": 0,
  "maxCapacity": 4,
  "scalingRatio": 0,
  "minPrice": 0.5,
  "maxPrice": 2,
  "canUseOndemand": false,
  "canUseSpot": true,
  "instanceTypes": [
    {
      "instanceType": "m3.2xlarge",
      "capacity": 1,
      "utility": 1,
      "secrets": {},
      "scopes": [],
      "userData": {},
      "launchSpec": {}
    }
  ],
  "regions": [
    {
      "region": "us-west-2",
      "secrets": {},
      "scopes": [],
      "userData": {},
      "launchSpec": {
        "ImageId": "ami-db657feb"
      }
    }
  ],
  "lastModified": "2015-09-30T10:15:30.349Z",
  "userData": {},
  "launchSpec": {
    "SecurityGroups": [
      "rdp-only"
    ]
  },
  "secrets": {},
  "scopes": [
    "*"
  ]
}
{% endhighlight %}

Not everybody has permission to create worker types - but there again, you only really need to do this if you are:

* using Windows (or anything else non-linux)
* not able to use an existing worker type

If you would like to create a new Worker Type, please contact the taskcluster team on `irc.mozilla.org` in `#taskcluster` channel.

The Worker Type above boils down to some AWS hardware specs, and an ImageId `ami-db657feb`. But where did this come from?

Generating the AMI for the Worker Type
--------------------------------------

It is a Windows 2012 R2 AMI, and it was generated with [this](https://hg.mozilla.org/try/file/fc4b30cc56fb/testing/taskcluster/worker_types/win2012r2) code checked in to the try branch. This is not automatically run, but is checked in for reference purposes.

Here is the code. The first is a script that creates the AMI:

{% highlight bash %}
#!/bin/bash -exv

# cd into directory containing script...
cd "$(dirname "${0}")"

# generate a random slugid for aws client token...
# you need either go installed (https://golang.org/) and $GOPATH configured to run this,
# or alternatively download the 'slug' binary; see
# http://taskcluster.github.io/slugid-go/#installing-command-line-tool
go get github.com/taskcluster/slugid-go/slug
SLUGID=$("${GOPATH}/bin/slug")

# aws cli docs lie, they say userdata must be base64 encoded, but cli encodes for you, so just cat it...
USER_DATA="$(cat aws_userdata)"

# create base ami, and apply user-data
# filter output, to get INSTANCE_ID
# N.B.: ami-4dbcb67d referenced below is *the* Windows 2012 Server R2 ami offered by Amazon in us-west-2 - it is nothing we have made
# note, you'll need aws tool installed, access to the taskcluster AWS account, and your own private key file
INSTANCE_ID="$(aws --region us-west-2 ec2 run-instances --image-id ami-4dbcb67d --key-name pmoore-oregan-us-west-2 --security-groups "RDP only" --user-data "${USER_DATA}" --instance-type c4.2xlarge --block-device-mappings DeviceName=/dev/sda1,Ebs='{VolumeSize=75,DeleteOnTermination=true,VolumeType=gp2}' --instance-initiated-shutdown-behavior terminate --client-token "${SLUGID}" | sed -n 's/^ *"InstanceId": "\(.*\)", */\1/p')"

# sleep an hour, the installs take forever...
sleep 3600

# now capture the AMI - feel free to change the tags
IMAGE_ID="$(aws --region us-west-2 ec2 create-image --instance-id "${INSTANCE_ID}" --name "win2012r2 mozillabuild pmoore version ${SLUGID}" --description "firefox desktop builds on windows - taskcluster worker - version ${SLUGID}" | sed -n 's/^ *"ImageId": *"\(.*\)" *$/\1/p')"

# TODO: now update worker type...
# You must update the existing win2012r2 worker type with the new ami id generated ($IMAGE_ID var above)
# At the moment this is a manual step! It can be automated following the docs:
# http://docs.taskcluster.net/aws-provisioner/api-docs/#workerType
# http://docs.taskcluster.net/aws-provisioner/api-docs/#updateWorkerType

echo "Worker type ami to be used: '${IMAGE_ID}' - don't forget to update https://tools.taskcluster.net/aws-provisioner/#win2012r2/edit"' !!!'
{% endhighlight %}

This script works by exploiting the fact that when you spawn a Windows instance in AWS, using one of the AMIs that Amazon provides, you can include a Powershell snippet for additional setup. This gets executed automatically when you spawn the instance.

So we simply spawn an instance, passing through this powershell snippet, and then wait. A LONG time (an hour). And then we snapshot the image, and we have our new AMI. Simple!

Here is the Powershell snippet that it uses:

{% highlight powershell %}
<powershell>

# needed for making http requests
$client = New-Object system.net.WebClient
$shell = new-object -com shell.application

# utility function to download a zip file and extract it
function Expand-ZIPFile($file, $destination, $url)
{
    $client.DownloadFile($url, $file)
    $zip = $shell.NameSpace($file)
    foreach($item in $zip.items())
    {
        $shell.Namespace($destination).copyhere($item)
    }
}

# allow powershell scripts to run
Set-ExecutionPolicy Unrestricted -Force -Scope Process

# install chocolatey package manager
Invoke-Expression ($client.DownloadString('https://chocolatey.org/install.ps1'))

# download mozilla-build installer
$client.DownloadFile("https://api.pub.build.mozilla.org/tooltool/sha512/03b4ca2bebede21a29f739165030bfc7058a461ffe38113452e976193e382d3ba6df8a48ac843b70429e23481e6327f43c86ffd88e4ce16263d072ef7e14e692", "C:\MozillaBuildSetup-2.0.0.exe")

# run mozilla-build installer in silent (/S) mode
$p = Start-Process "C:\MozillaBuildSetup-2.0.0.exe" -ArgumentList "/S" -wait -NoNewWindow -PassThru -RedirectStandardOutput "C:\MozillaBuild-2.0.0_install.log" -RedirectStandardError "C:\MozillaBuild-2.0.0_install.err"

# install Windows SDK 8.1
choco install -y windows-sdk-8.1

# install Visual Studio community edition 2013
choco install -y visualstudiocommunity2013
# $client.DownloadFile("https://go.microsoft.com/fwlink/?LinkId=532495&clcid=0x409", "C:\vs_community.exe")

# install June 2010 DirectX SDK for compatibility with Win XP
$client.DownloadFile("http://download.microsoft.com/download/A/E/7/AE743F1F-632B-4809-87A9-AA1BB3458E31/DXSDK_Jun10.exe", "C:\DXSDK_Jun10.exe")

# prerequisite for June 2010 DirectX SDK is to install ".NET Framework 3.5 (includes .NET 2.0 and 3.0)"
Install-WindowsFeature NET-Framework-Core -Restart

# now run DirectX SDK installer
$p = Start-Process "C:\DXSDK_Jun10.exe" -ArgumentList "/U" -wait -NoNewWindow -PassThru -RedirectStandardOutput C:\directx_sdk_install.log -RedirectStandardError C:\directx_sdk_install.err

# install PSTools
md "C:\PSTools"
Expand-ZIPFile -File "C:\PSTools\PSTools.zip" -Destination "C:\PSTools" -Url "https://download.sysinternals.com/files/PSTools.zip"

# install nssm
Expand-ZIPFile -File "C:\nssm-2.24.zip" -Destination "C:\" -Url "http://www.nssm.cc/release/nssm-2.24.zip"

# download generic-worker
md "C:\generic-worker"
$client.DownloadFile("https://github.com/taskcluster/generic-worker/releases/download/v1.0.12/generic-worker-windows-amd64.exe", "C:\generic-worker\generic-worker.exe")

# enable DEBUG logs for generic-worker install
$env:DEBUG = "*"

# install generic-worker
$p = Start-Process "C:\generic-worker\generic-worker.exe" -ArgumentList "install --config C:\\generic-worker\\generic-worker.config" -wait -NoNewWindow -PassThru -RedirectStandardOutput C:\generic-worker\install.log -RedirectStandardError C:\generic-worker\install.err

# add extra config needed
$config = [System.Convert]::FromBase64String("UEsDBAoAAAAAAA2hN0cIOIW2JwAAACcAAAAJAAAAZ2FwaS5kYXRhQUl6YVN5RC1zLW1YTDRtQnpGN0tNUmtoVENJYkcyUktuUkdYekpjUEsDBAoAAAAAACehN0cVjoCGIAAAACAAAAAVAAAAY3Jhc2gtc3RhdHMtYXBpLnRva2VuODhmZjU3ZDcxMmFlNDVkYmJlNDU3NDQ1NWZjYmNjM2VQSwMECgAAAAAANKE3RxYFa6ViAAAAYgAAABQAAABnb29nbGUtb2F1dGgtYXBpLmtleTE0NzkzNTM0MzU4Mi1qZmwwZTBwc2M3a2gxbXV0MW5mdGI3ZGUwZjFoMHJvMC5hcHBzLmdvb2dsZXVzZXJjb250ZW50LmNvbSBLdEhDRkNjMDlsdEN5SkNqQ3dIN1pKd0cKUEsDBAoAAAAAAEShN0ctdLepZAAAAGQAAAAYAAAAZ29vZ2xlLW9hdXRoLWFwaS5rZXlfYmFr77u/MTQ3OTM1MzQzNTgyLWpmbDBlMHBzYzdraDFtdXQxbmZ0YjdkZTBmMWgwcm8wLmFwcHMuZ29vZ2xldXNlcmNvbnRlbnQuY29tIEt0SENGQ2MwOWx0Q3lKQ2pDd0g3Wkp3R1BLAwQKAAAAAABYoTdHJ3EEFiQAAAAkAAAADwAAAG1vemlsbGEtYXBpLmtleTNiNGQyN2RkLTcwM2QtNDA5NC04Mzk4LTRkZTJjNzYzNTA1YVBLAwQKAAAAAABkoTdHMi/H2yQAAAAkAAAAHgAAAG1vemlsbGEtZGVza3RvcC1nZW9sb2MtYXBpLmtleTdlNDBmNjhjLTc5MzgtNGM1ZC05Zjk1LWU2MTY0N2MyMTNlYlBLAwQKAAAAAABxoTdHJ3EEFiQAAAAkAAAAHQAAAG1vemlsbGEtZmVubmVjLWdlb2xvYy1hcGkua2V5M2I0ZDI3ZGQtNzAzZC00MDk0LTgzOTgtNGRlMmM3NjM1MDVhUEsDBBQAAAAIAHyhN0fa715hagAAAHMAAAANAAAAcmVsZW5nYXBpLnRva0ut9MpIck/O9M/08gyt8jT0y/Sy1Eut9CpINvYFCVZGhnhm+jh7Faa4Z4P4Br4QvkFqhCOIX56ca5CZFqiXU5VoWeaSm20S6eblE+rpXJDiFxoRVBphnFFZUmrpkphd7m4aVWXsFxQeCABQSwECHgMKAAAAAAANoTdHCDiFticAAAAnAAAACQAAAAAAAAABAAAApIEAAAAAZ2FwaS5kYXRhUEsBAh4DCgAAAAAAJ6E3RxWOgIYgAAAAIAAAABUAAAAAAAAAAQAAAKSBTgAAAGNyYXNoLXN0YXRzLWFwaS50b2tlblBLAQIeAwoAAAAAADShN0cWBWulYgAAAGIAAAAUAAAAAAAAAAEAAACkgaEAAABnb29nbGUtb2F1dGgtYXBpLmtleVBLAQIeAwoAAAAAAEShN0ctdLepZAAAAGQAAAAYAAAAAAAAAAEAAACkgTUBAABnb29nbGUtb2F1dGgtYXBpLmtleV9iYWtQSwECHgMKAAAAAABYoTdHJ3EEFiQAAAAkAAAADwAAAAAAAAABAAAApIHPAQAAbW96aWxsYS1hcGkua2V5UEsBAh4DCgAAAAAAZKE3RzIvx9skAAAAJAAAAB4AAAAAAAAAAQAAAKSBIAIAAG1vemlsbGEtZGVza3RvcC1nZW9sb2MtYXBpLmtleVBLAQIeAwoAAAAAAHGhN0cncQQWJAAAACQAAAAdAAAAAAAAAAEAAACkgYACAABtb3ppbGxhLWZlbm5lYy1nZW9sb2MtYXBpLmtleVBLAQIeAxQAAAAIAHyhN0fa715hagAAAHMAAAANAAAAAAAAAAEAAACkgd8CAAByZWxlbmdhcGkudG9rUEsFBgAAAAAIAAgAEQIAAHQDAAAAAA==")
md "C:\builds"
Set-Content -Path "C:\builds\config.zip" -Value $config -Encoding Byte
$zip = $shell.NameSpace("C:\builds\config.zip")
foreach($item in $zip.items())
{
    $shell.Namespace("C:\builds").copyhere($item)
}
rm "C:\builds\config.zip"

# initial clone of mozilla-central
$p = Start-Process "C:\mozilla-build\python\python.exe" -ArgumentList "C:\mozilla-build\python\Scripts\hg clone -u null https://hg.mozilla.org/mozilla-central C:\gecko" -wait -NoNewWindow -PassThru -RedirectStandardOutput "C:\hg_initial_clone.log" -RedirectStandardError "C:\hg_initial_clone.err"
{% endhighlight %}

Hopefully this Powershell script is quite self-explanatory. It installs the required build tool chains for building Firefox Desktop, and then installs the parts it needs for running the Generic Worker on this instance. It sets up some additional config that is needed by the build process, and then takes an initial clone of mozilla-central, as an optimisation, so that future jobs only need to pull changes since the image was created.

The caching strategy is to have a clone of mozilla-central live under `C:\gecko`, which is updated with an `hg pull` from mozilla central each time a job runs. Then when a task needs to pull from try, it is only ever a few commits behind, and should pull updates very quickly.

Defining Tasks
--------------

Once we have our AMI created, and we've published our Worker Type, we need to submit tasks to get the Provisioner to spawn instances in AWS, and execute our tasks.

The next piece of the puzzle is working out how to get these jobs added to Try. Again, luckily for us, this is just a matter of in-tree config.

For this, most of the magic exists in [testing/taskcluster/tasks/builds/firefox_windows_base.yml](https://hg.mozilla.org/try/file/fc4b30cc56fb/testing/taskcluster/tasks/builds/firefox_windows_base.yml):

{% highlight yaml %}
$inherits:
  from: 'tasks/windows_build.yml'
  variables:
    build_product: 'firefox'

task:
  metadata:
    name: "[TC] Firefox {{ "{{arch"}}}} ({{ "{{build_type"}}}})"
    description: Firefox {{ "{{arch"}}}} {{ "{{build_type"}}}}

  payload:
    env:
      ExtensionSdkDir: "C:\\Program Files (x86)\\Microsoft SDKs\\Windows\\v8.1\\ExtensionSDKs"
      Framework40Version: "v4.0"
      FrameworkDir: "C:\\Windows\\Microsoft.NET\\Framework64"
      FrameworkDIR64: "C:\\Windows\\Microsoft.NET\\Framework64"
      FrameworkVersion: "v4.0.30319"
      FrameworkVersion64: "v4.0.30319"
      FSHARPINSTALLDIR: "C:\\Program Files (x86)\\Microsoft SDKs\\F#\\3.1\\Framework\\v4.0\\"
      INCLUDE: "C:\\Program Files (x86)\\Microsoft Visual Studio 12.0\\VC\\INCLUDE;C:\\Program Files (x86)\\Microsoft Visual Studio 12.0\\VC\\ATLMFC\\INCLUDE;C:\\Program Files (x86)\\Windows Kits\\8.1\\include\\shared;C:\\Program Files (x86)\\Windows Kits\\8.1\\include\\um;C:\\Program Files (x86)\\Windows Kits\\8.1\\include\\winrt;"
      MOZBUILD_STATE_PATH: "C:\\Users\\Administrator\\.mozbuild"
      MOZ_MSVCVERSION: "12"
      MOZ_MSVCYEAR: "2013"
      MOZ_TOOLS: "C:\\mozilla-build\\moztools-x64"
      MSVCKEY: "HKLM\\SOFTWARE\\Wow6432Node\\Microsoft\\VisualStudio\\12.0\\Setup\\VC"
      SDKDIR: "C:\\Program Files (x86)\\Windows Kits\\8.1\\"
      SDKMINORVER: "1"
      SDKPRODUCTKEY: "HKLM\\SOFTWARE\\Microsoft\\Windows Kits\\Installed Products"
      SDKROOTKEY: "HKLM\\SOFTWARE\\Microsoft\\Windows Kits\\Installed Roots"
      SDKVER: "8"
      VCDIR: "C:\\Program Files (x86)\\Microsoft Visual Studio 12.0\\VC\\"
      VCINSTALLDIR: "C:\\Program Files (x86)\\Microsoft Visual Studio 12.0\\VC\\"
      VisualStudioVersion: "12.0"
      VSINSTALLDIR: "C:\\Program Files (x86)\\Microsoft Visual Studio 12.0\\"
      WIN64: "1"
      WIN81SDKKEY: "{5247E16E-BCF8-95AB-1653-B3F8FBF8B3F1}"
      WINCURVERKEY: "HKLM\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion"
      WindowsSdkDir: "C:\\Program Files (x86)\\Windows Kits\\8.1\\"
      WindowsSDK_ExecutablePath_x64: "C:\\Program Files (x86)\\Microsoft SDKs\\Windows\\v8.1A\\bin\\NETFX 4.5.1 Tools\\x64\\"
      WindowsSDK_ExecutablePath_x86: "C:\\Program Files (x86)\\Microsoft SDKs\\Windows\\v8.1A\\bin\\NETFX 4.5.1 Tools\\"
      MACHTYPE: "i686-pc-msys"
      MAKE_MODE: "unix"
      MOZBUILDDIR: "C:\\mozilla-build"
      MOZILLABUILD: "C:\\mozilla-build"
      MOZ_AUTOMATION: "1"
      MOZ_BUILD_DATE: "19770819000000"
      MOZ_CRASHREPORTER_NO_REPORT: "1"
      MSYSTEM: "MINGW32"

    command:
      - "time /t && set"
      - "time /t && hg -R C:\\gecko pull"
      - "time /t && hg clone C:\\gecko src"
      - "time /t && mkdir public\\build"
      - "time /t && set UPLOAD_HOST=localhost"
      - "time /t && set UPLOAD_PATH=%CD%\\public\\build"
      - "time /t && cd src"
      - "time /t && hg pull -r %GECKO_HEAD_REV% -u %GECKO_HEAD_REPOSITORY%"
      - "time /t && set MOZCONFIG=%CD%\\{{ "{{mozconfig"}}}}"
      - "time /t && set SRCSRV_ROOT=%GECKO_HEAD_REPOSITORY%"
      - "time /t && C:\\mozilla-build\\msys\\bin\\bash --login %CD%\\mach build"

    artifacts:

      # In the next few days I plan to provide support for directory artifacts,
      # so this explicit list will no longer be needed, and you can specify the
      # following:
      # -
      #   type: "directory"
      #   path: "public\\build"
      #   expires: '{{ "{{#from_now"}}}}1 year{{ "{{/from_now"}}}}'
      #
      #  This will be done in early October 2015. See
      #  https://bugzilla.mozilla.org/show_bug.cgi?id=1209901

      -
        type: "file"
        path: "public\\build\\firefox-43.0a1.en-US.{{ "{{arch"}}}}.checksums"
        expires: '{{ "{{#from_now"}}}}1 year{{ "{{/from_now"}}}}'
      -
        type: "file"
        path: "public\\build\\firefox-43.0a1.en-US.{{ "{{arch"}}}}.common.tests.zip"
        expires: '{{ "{{#from_now"}}}}1 year{{ "{{/from_now"}}}}'
      -
        type: "file"
        path: "public\\build\\firefox-43.0a1.en-US.{{ "{{arch"}}}}.cppunittest.tests.zip"
        expires: '{{ "{{#from_now"}}}}1 year{{ "{{/from_now"}}}}'
      -
        type: "file"
        path: "public\\build\\firefox-43.0a1.en-US.{{ "{{arch"}}}}.crashreporter-symbols.zip"
        expires: '{{ "{{#from_now"}}}}1 year{{ "{{/from_now"}}}}'
      -
        type: "file"
        path: "public\\build\\firefox-43.0a1.en-US.{{ "{{arch"}}}}.json"
        expires: '{{ "{{#from_now"}}}}1 year{{ "{{/from_now"}}}}'
      -
        type: "file"
        path: "public\\build\\firefox-43.0a1.en-US.{{ "{{arch"}}}}.mochitest.tests.zip"
        expires: '{{ "{{#from_now"}}}}1 year{{ "{{/from_now"}}}}'
      -
        type: "file"
        path: "public\\build\\firefox-43.0a1.en-US.{{ "{{arch"}}}}.mozinfo.json"
        expires: '{{ "{{#from_now"}}}}1 year{{ "{{/from_now"}}}}'
      -
        type: "file"
        path: "public\\build\\firefox-43.0a1.en-US.{{ "{{arch"}}}}.reftest.tests.zip"
        expires: '{{ "{{#from_now"}}}}1 year{{ "{{/from_now"}}}}'
      -
        type: "file"
        path: "public\\build\\firefox-43.0a1.en-US.{{ "{{arch"}}}}.talos.tests.zip"
        expires: '{{ "{{#from_now"}}}}1 year{{ "{{/from_now"}}}}'
      -
        type: "file"
        path: "public\\build\\firefox-43.0a1.en-US.{{ "{{arch"}}}}.txt"
        expires: '{{ "{{#from_now"}}}}1 year{{ "{{/from_now"}}}}'
      -
        type: "file"
        path: "public\\build\\firefox-43.0a1.en-US.{{ "{{arch"}}}}.web-platform.tests.zip"
        expires: '{{ "{{#from_now"}}}}1 year{{ "{{/from_now"}}}}'
      -
        type: "file"
        path: "public\\build\\firefox-43.0a1.en-US.{{ "{{arch"}}}}.xpcshell.tests.zip"
        expires: '{{ "{{#from_now"}}}}1 year{{ "{{/from_now"}}}}'
      -
        type: "file"
        path: "public\\build\\firefox-43.0a1.en-US.{{ "{{arch"}}}}.zip"
        expires: '{{ "{{#from_now"}}}}1 year{{ "{{/from_now"}}}}'
      -
        type: "file"
        path: "public\\build\\host\\bin\\mar.exe"
        expires: '{{ "{{#from_now"}}}}1 year{{ "{{/from_now"}}}}'
      -
        type: "file"
        path: "public\\build\\host\\bin\\mbsdiff.exe"
        expires: '{{ "{{#from_now"}}}}1 year{{ "{{/from_now"}}}}'
      -
        type: "file"
        path: "public\\build\\install\\sea\\firefox-43.0a1.en-US.{{ "{{arch"}}}}.installer.exe"
        expires: '{{ "{{#from_now"}}}}1 year{{ "{{/from_now"}}}}'
      -
        type: "file"
        path: "public\\build\\jsshell-{{ "{{arch"}}}}.zip"
        expires: '{{ "{{#from_now"}}}}1 year{{ "{{/from_now"}}}}'
      -
        type: "file"
        path: "public\\build\\test_packages.json"
        expires: '{{ "{{#from_now"}}}}1 year{{ "{{/from_now"}}}}'
      -
        type: "file"
        path: "public\\build\\{{ "{{arch"}}}}\\xpi\\firefox-43.0a1.en-US.langpack.xpi"
        expires: '{{ "{{#from_now"}}}}1 year{{ "{{/from_now"}}}}'

  extra:
    treeherderEnv:
      - production
      - staging
    treeherder:
      groupSymbol: "tc"
      groupName: Submitted by taskcluster
      machine:
        # from https://github.com/mozilla/treeherder/blob/9263d8432642c2ca9f68b301250af0ffbec27d83/ui/js/values.js#L3
        platform: {{ "{{platform"}}}}

    # Rather then enforcing particular conventions we require that all build
    # tasks provide the "build" extra field to specify where the build and tests
    # files are located.
    locations:
      build: "src/{{ "{{object_dir"}}}}/dist/bin/firefox.exe"
      tests: "src/{{ "{{object_dir"}}}}/all-tests.json"
{% endhighlight %}

Reading through this, you see that with the exception of knowing the value of a few parameters (`{{ "{{object_dir"}}}}`, `{{ "{{platform"}}}}`, `{{ "{{arch"}}}}`, `{{ "{{build_type"}}}}`, `{{ "{{mozconfig"}}}}`), the full set of steps that a Windows build of Firefox Desktop requires on the Worker Type we created above. In other words, you see the full system setup in the Worker Type definition, and the full set of task steps in this Task Definition - so now you know as much as I do about how to build Firefox Desktop on Windows. It all exists in-tree, and is transparent to developers.

So where do these parameters come from? Well, this is just the _base_ config - we define opt and debug builds for win32 and win64 architectures. These live [here]:

* [win32 opt](https://hg.mozilla.org/try/file/fc4b30cc56fb/testing/taskcluster/tasks/builds/firefox_win32_opt.yml)
* [win32 debug](https://hg.mozilla.org/try/file/fc4b30cc56fb/testing/taskcluster/tasks/builds/firefox_win32_debug.yml)
* [win64 opt](https://hg.mozilla.org/try/file/fc4b30cc56fb/testing/taskcluster/tasks/builds/firefox_win64_opt.yml)
* [win64 debug](https://hg.mozilla.org/try/file/fc4b30cc56fb/testing/taskcluster/tasks/builds/firefox_win64_debug.yml)

Here I will illustrate just one of them, the win32 debug build config:

{% highlight yaml %}
$inherits:
  from: 'tasks/builds/firefox_windows_base.yml'
  variables:
    build_type: 'debug'
    arch: 'win32'
    platform: 'windowsxp'
    object_dir: 'obj-i686-pc-mingw32'
    mozconfig: 'browser\\config\\mozconfigs\\win32\\debug'
task:
  extra:
    treeherder:
      collection:
        debug: true
  payload:
    env:
      CommandPromptType: "Cross"
      LIB: "C:\\Program Files (x86)\\Microsoft Visual Studio 12.0\\VC\\LIB;C:\\Program Files (x86)\\Microsoft Visual Studio 12.0\\VC\\ATLMFC\\LIB;C:\\Program Files (x86)\\Windows Kits\\8.1\\lib\\winv6.3\\um\\x86;"
      LIBPATH: "C:\\Windows\\Microsoft.NET\\Framework64\\v4.0.30319;C:\\Program Files (x86)\\Microsoft Visual Studio 12.0\\VC\\LIB;C:\\Program Files (x86)\\Microsoft Visual Studio 12.0\\VC\\ATLMFC\\LIB;C:\\Program Files (x86)\\Windows Kits\\8.1\\References\\CommonConfiguration\\Neutral;C:\\Program Files (x86)\\Microsoft SDKs\\Windows\\v8.1\\ExtensionSDKs\\Microsoft.VCLibs\\12.0\\References\\CommonConfiguration\\neutral;"
      MOZ_MSVCBITS: "32"
      Path: "C:\\Program Files (x86)\\Microsoft Visual Studio 12.0\\Common7\\IDE\\CommonExtensions\\Microsoft\\TestWindow;C:\\Program Files (x86)\\MSBuild\\12.0\\bin\\amd64;C:\\Program Files (x86)\\Microsoft Visual Studio 12.0\\VC\\BIN\\amd64_x86;C:\\Program Files (x86)\\Microsoft Visual Studio 12.0\\VC\\BIN\\amd64;C:\\Windows\\Microsoft.NET\\Framework64\\v4.0.30319;C:\\Program Files (x86)\\Microsoft Visual Studio 12.0\\VC\\VCPackages;C:\\Program Files (x86)\\Microsoft Visual Studio 12.0\\Common7\\IDE;C:\\Program Files (x86)\\Microsoft Visual Studio 12.0\\Common7\\Tools;C:\\Program Files (x86)\\HTML Help Workshop;C:\\Program Files (x86)\\Microsoft Visual Studio 12.0\\Team Tools\\Performance Tools\\x64;C:\\Program Files (x86)\\Microsoft Visual Studio 12.0\\Team Tools\\Performance Tools;C:\\Program Files (x86)\\Windows Kits\\8.1\\bin\\x64;C:\\Program Files (x86)\\Windows Kits\\8.1\\bin\\x86;C:\\Program Files (x86)\\Microsoft SDKs\\Windows\\v8.1A\\bin\\NETFX 4.5.1 Tools\\x64\\;C:\\Windows\\System32;C:\\Windows;C:\\Windows\\System32\\Wbem;C:\\mozilla-build\\moztools-x64\\bin;C:\\mozilla-build\\7zip;C:\\mozilla-build\\info-zip;C:\\mozilla-build\\kdiff3;C:\\mozilla-build\\mozmake;C:\\mozilla-build\\nsis-3.0b1;C:\\mozilla-build\\nsis-2.46u;C:\\mozilla-build\\python;C:\\mozilla-build\\python\\Scripts;C:\\mozilla-build\\upx391w;C:\\mozilla-build\\wget;C:\\mozilla-build\\yasm"
      Platform: "X86"
      PreferredToolArchitecture: "x64"
      TOOLCHAIN: "64-bit cross-compile"
{% endhighlight %}

This file above has defined those parameters, and provided some more task specific config too, which overlays the base config we saw before.

Getting the new tasks added to Try pushes

This involved adding `win32` and `win64` as build platforms in `testing/taskcluster/tasks/branches/base_job_flags.yml` (previsouly taskcluster was not running any tasks for these platforms):

{% highlight yaml %}
---
# List of all possible flags for each category of tests used in the case where
# "all" is specified.
flags:
  aliases:
    mochitests: mochitest

  builds:
    - emulator
    - emulator-jb
    - emulator-kk
    - emulator-x86-kk
    .....
    .....
    .....
    - android-api-11
    - linux64
    - macosx64
    - win32   ########## <---- added here
    - win64   ########## <---- added here

  tests:
    - cppunit
    - crashtest
    - crashtest-ipc
    - gaia-build
    .....
    .....
    .....
{% endhighlight %}

And then associating these new task definitions we just created, to these new build platforms. This is done in `testing/taskcluster/tasks/branches/try/job_flags.yml`:

{%highlight yaml %}
---
# For complete sample of all build and test jobs,
# see <gecko>/testing/taskcluster/tasks/job_flags.yml

$inherits:
  from: tasks/branches/base_job_flags.yml

# Flags specific to this branch
flags:
  post-build:
    - upload-symbols

builds:
  win32:
    platforms:
      - win32
    types:
      opt:
        task: tasks/builds/firefox_win32_opt.yml
      debug:
        task: tasks/builds/firefox_win32_debug.yml
  win64:
    platforms:
      - win64
    types:
      opt:
        task: tasks/builds/firefox_win64_opt.yml
      debug:
        task: tasks/builds/firefox_win64_debug.yml
  linux64_gecko:
    platforms:
      - b2g
    types:
      opt:
    .....
    .....
    .....
{% endhighlight %}
 
Summary
=======

The above hopefully has given you a taste for what you can do yourself in TaskCluster, and specifically in Gecko, regarding setting up new jobs. By following this guide, you too should be able to schedule Windows jobs in Taskcluster, including try jobs for [Gecko](https://developer.mozilla.org/en-US/docs/Mozilla/Gecko) projects.

For more information about TaskCluster, see [docs.taskcluster.net](https://docs.taskcluster.net).
