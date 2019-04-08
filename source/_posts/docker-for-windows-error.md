---
title: Docker for Windows 出现 Could not save daemon configuration to C:\ProgramData\Docker\config\daemon.json
date: 2019-04-08 16:09:46
tags: Docker
---

安装好 Docker for Windows 后，点击 Switch to Windows containers 可能会出现错误：

![](docker-for-windows-error/2019-04-08-16-42-33.png)
出现这种情况是因为没有对 `C:\ProgramData\Docker\config\daemon.json` 的读写权限，这时候只需要进入目录 `C:\ProgramData\Docker`，
出现下面提示时，点击**继续**，获得权限即可

![](docker-for-windows-error/2019-04-08-16-44-24.png)