---
author: "Alfero Chingono"
title: "Resolving Azure Devops Fastlane Error 'An exception has occurred: issuerId is required'"
date: 2022-10-24T18:06:36Z
draft: false
description: "When deploying iOS apps with Azure DevOps, you may run into the error message: 'An exception has occurred: issuerId is required'. Changing the Agent Specification to macOS-12 will resolve the issue."
slug: resolving-azure-devops-fastlane-error-issuerid-required
tags: [
"azure",
"devops",
"cd",
"fastlane",
"ios",
"testflight"
]
categories: [
"Development",
"Deployment",
"Release"
]
image: "cover.png"
---

While releasing an updated version of a mobile application to the App Store Test Flight track, I noticed the following error in the logs:

```log
2022-10-24T21:31:36.1408460Z [21:31:36]: There was a general exception while executing
2022-10-24T21:31:36.1409440Z An exception has occurred: issuerId is required
2022-10-24T21:31:36.1410980Z Return status of iTunes Transporter was 1: There was a general exception while executing
2022-10-24T21:31:36.1411520Z \nAn exception has occurred: issuerId is required
2022-10-24T21:31:36.1412610Z The call to the iTMSTransporter completed with a non-zero exit status: 1. This indicates a failure.
2022-10-24T21:31:36.1567410Z 
2022-10-24T21:31:36.1568750Z [!] Error uploading ipa file: 
2022-10-24T21:31:36.1569190Z  
2022-10-24T21:31:36.1893760Z ##[error]Error: The process '/usr/local/lib/ruby/gems/2.7.0/bin/fastlane' failed with exit code 1
```

A quick search for `fastlane +"An exception has occurred: issuerId is required"` led me to this issue on GitHub:

[ [pilot] fails to upload build to TestFlight using api key after iTMSTransporter auto updated to version 3.0.0 with An exception has occurred: issuerId is required error](https://github.com/fastlane/fastlane/issues/20741)

One particular [comment by Tobias Totzek](https://github.com/fastlane/fastlane/issues/20741#issuecomment-1285291626) provided the exact answer to my problem. I changed the Agent Specification from `macos-latest` to `macOS-12` and problem solved.

Credits:  
[Tobias Totzek](https://github.com/honkmaster)  
