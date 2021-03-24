# 开发指南

## BusyBox
Magisk附带了功能完整的BusyBox二进制文件(包括对SELinux的完整支持)。执行文件位于`/data/adb/magisk/busybox`。Magisk的BusyBox支持运行时可切换的“ASH Standalone Shell Mode(ASH独立Shell模式)”。这种独立模式的意思是，在`ash`shell的中的BusyBox运行时，无论设置为什么`PATH`，每个命令都将直接使用BusyBox中的应用(子命令)。例如，就像`ls`, `rm`, `chmod`命令一样。它们将**不再**使用用`PATH` (在Android中，默认为`/system/bin/ls`, `/system/bin/rm`, and `/system/bin/chmod` respectively)，而是直接调用Magisk内部的BusyBox应用(子命令)。这样可以确保脚本始终在可预测的环境中运行，并且无论运行在哪个Android版本上，始终具有完整的命令集。要强制命令*不使用*BusyBox，必须使用完整路径调用可执行文件。

在启用了`ash`的独立模式的情况下，Magisk中运行的每个单独的shell脚本都将在内部的BusyBox的shell中执行。对于与第三方开发者而言，这包括所有引导脚本和模块安装脚本。

对于那些想在Magisk之外使用此“独立模式”功能的人，有两种启用它的方法:

1. 将环境变量`ASH_STANDALONE`设置为`1`<br>示例: `ASH_STANDALONE=1 /data/adb/magisk/busybox sh <script>`
2. 切换命令选项:<br>`/data/adb/magisk/busybox sh -o standalone <script>`

为确保`sh`执行的shell也都以独立模式运行，推荐使用1方案(这是Magisk和Magisk应用程序内部使用的方法)，因为环境变量一直继承到子进程。


## Magisk模块
一个Magisk模块的文件在`/data/adb/modules`具有以下结构:

```
/data/adb/modules
├── .
├── .
|
├── $MODID                  <--- 该文件夹以模块的ID命名
│   │
│   │      *** 模块信息 ***
│   │
│   ├── module.prop         <--- 该文件存储模块的基本信息
│   │
│   │      *** 主要内容 ***
│   │
│   ├── system              <--- 如果skip_mount不存在，则将挂载此文件夹
│   │   ├── ...
│   │   ├── ...
│   │   └── ...
│   │
│   │      *** 状态标识 ***
│   │
│   ├── skip_mount          <--- 如果存在，Magisk将不会挂载你的system文件夹
│   ├── disable             <--- 如果存在，该模块将被禁用
│   ├── remove              <--- 如果存在，该模块将在下次重新启动时被删除
│   │
│   │      *** 可选文件 ***
│   │
│  ├── post-fs-data.sh     <--- 该脚本将在post-fs-data模式下执行
│  ├── service.sh          <--- 该脚本将在late_start service模式执行
|   ├── uninstall.sh        <--- 当Magisk删除您的模块时，将执行此脚本
│  ├── system.prop         <--- 该文件中的properties将通过resetprop作为系统properties加载
│  ├── sepolicy.rule       <--- 添加自定义的sepolicy规则
│  │
│   │      *** 自动生成，请勿手动创建或修改 ***
│  │
│  ├── vendor              <--- 指向$MODID/system/vendor的链接
│  ├── product             <--- 指向$MODID/system/product的链接
│  ├── system_ext          <--- 指向$MODID/system/system_ext的链接
│   │
│   │      *** 允许任何其他文件/文件夹 ***
│   │
│  ├── ...
│  └── ...
|
├── another_module
│  ├── .
│  └── .
├── .
├── .
```


#### module.prop

这是**标准的**`module.prop`格式

```
id=<string>
name=<string>
version=<string>
versionCode=<int>
author=<string>
description=<string>
```
- `id`必须匹配以下正则表达式:`^[a-zA-Z][a-zA-Z0-9._-]+$`<br>
ex: ✓ `a_module`, ✓ `a.module`, ✓ `module-101`, ✗ `a module`, ✗ `1_module`, ✗ `-a-module`<br>
这是模块的**唯一标识ID**。发布后，不应改变它。
- `versionCode`必须为**整数**。这用于比较版本。
- 上面未提及的其他字符串可以是任何**单行**字符串。
- 请确保使用`UNIX (LF)`类型换行符，而不是`Windows (CR+LF)`或`Macintosh (CR)`。

#### Shell脚本 (`*.sh`)
请阅读[启动脚本](#boot-scripts)了解`post-fs-data.sh`与`service.sh`之间的差异。对于大多数模块开发者，`service.sh`如果只需运行启动脚本那便足够了。

在所有的模块脚本中，请使用`MODDIR=${0%/*}`来获取模块的基本目录信息;千万**不要**在脚本中编码模块目录。

#### system.prop
该文件的格式与`build.prop`相同。每行包含`[key]=[value]`。

#### sepolicy.rule
If your module requires some additional sepolicy patches, please add those rules into this file. The module installer script and Magisk's daemon will make sure this file is copied to somewhere `magiskinit` can read pre-init to ensure these rules are injected properly.

Each line in this file will be treated as a policy statement. For more details about how a policy statement is formatted, please check [magiskpolicy](tools.md#magiskpolicy)'s documentation.

#### The `system` folder
All files you want Magisk to replace/inject for you should be placed in this folder. Please read through the [Magic Mount](details.md#magic-mount) section to understand how Magisk mount your files.

## Magisk Module Installer

A Magisk Module Installer is a Magisk Module packaged in a zip file that can be flashed in the Magisk app or custom recoveries such as TWRP. An installer has the same file structure as a Magisk module (please check the previous section for more info). The simplest Magisk Module Installer is just a Magisk Module packed in a zip file, with addition to the following files:

- `update-binary`: Download the latest [module_installer.sh](https://github.com/topjohnwu/Magisk/blob/master/scripts/module_installer.sh) and rename/copy that script as `update-binary`
- `updater-script`: This file should only contain the string `#MAGISK`

By default, `update-binary` will check/setup the environment, load utility functions, extract the module installer zip to where your module will be installed, and finally do some trivial tasks and cleanups, which should cover most simple modules' needs.

```
module.zip
│
├── META-INF
│   └── com
│       └── google
│           └── android
│               ├── update-binary      <--- The module_installer.sh you downloaded
│               └── updater-script     <--- Should only contain the string "#MAGISK"
│
├── customize.sh                       <--- (Optional, more details later)
│                                           This script will be sourced by update-binary
├── ...
├── ...  /* The rest of module's files */
│
```

#### Customization

If you need to customize the module installation process, optionally you can create a new script in the installer called `customize.sh`. This script will be *sourced* (not executed!) by `update-binary` after all files are extracted and default permissions and secontext are applied. This is very useful if your module includes different files based on ABI, or you need to set special permissions/secontext for some of your files (e.g. files in `/system/bin`).

If you need even more customization and prefer to do everything on your own, declare `SKIPUNZIP=1` in `customize.sh` to skip the extraction and applying default permissions/secontext steps. Be aware that by doing so, your `customize.sh` will then be responsible to install everything by itself.

#### `customize.sh` Environment

This script will run in Magisk's BusyBox `ash` shell with "Standalone Mode" enabled. The following variables and shell functions are available for convenience:

##### Variables
- `MAGISK_VER` (string): the version string of current installed Magisk (e.g. `v20.0`)
- `MAGISK_VER_CODE` (int): the version code of current installed Magisk (e.g. `20000`)
- `BOOTMODE` (bool): `true` if the module is being installed in the Magisk app
- `MODPATH` (path): the path where your module files should be installed
- `TMPDIR` (path): a place where you can temporarily store files
- `ZIPFILE` (path): your module's installation zip
- `ARCH` (string): the CPU architecture of the device. Value is either `arm`, `arm64`, `x86`, or `x64`
- `IS64BIT` (bool): `true` if `$ARCH` is either `arm64` or `x64`
- `API` (int): the API level (Android version) of the device (e.g. `21` for Android 5.0)

##### Functions

```
ui_print <msg>
    print <msg> to console
    Avoid using 'echo' as it will not display in custom recovery's console

abort <msg>
    print error message <msg> to console and terminate the installation
    Avoid using 'exit' as it will skip the termination cleanup steps

set_perm <target> <owner> <group> <permission> [context]
    if [context] is not set, the default is "u:object_r:system_file:s0"
    this function is a shorthand for the following commands:
       chown owner.group target
       chmod permission target
       chcon context target

set_perm_recursive <directory> <owner> <group> <dirpermission> <filepermission> [context]
    if [context] is not set, the default is "u:object_r:system_file:s0"
    for all files in <directory>, it will call:
       set_perm file owner group filepermission context
    for all directories in <directory> (including itself), it will call:
       set_perm dir owner group dirpermission context
```

##### Easy Replace
You can declare a list of folders you want to directly replace in the variable name `REPLACE`. The module installer script will pickup this variable and create `.replace` files for you. An example declaration:

```
REPLACE="
/system/app/YouTube
/system/app/Bloatware
"
```
The list above will result in the following files being created: `$MODPATH/system/app/YouTube/.replace` and `$MODPATH/system/app/Bloatware/.replace`

#### Notes

- When your module is downloaded with the Magisk app, `update-binary` will be **forcefully** replaced with the latest [`module_installer.sh`](https://github.com/topjohnwu/Magisk/blob/master/scripts/module_installer.sh) to ensure all installer uses up-to-date scripts. **DO NOT** try to add any custom logic in `update-binary` as it is pointless.
- Due to historical reasons, **DO NOT** add a file named `install.sh` in your module installer. That specific file was previously used and will be treated differently.
- **DO NOT** call `exit` at the end of `customize.sh`. The module installer would want to do finalizations.

## Submit Modules

You can submit a module to **Magisk-Module-Repo** so users can download your module directly in the Magisk app.

- Follow the instructions in the previous section to create a valid installer for your module.
- Create `README.md` (filename should be exactly the same) containing all info for your module. If you are not familiar with the Markdown syntax, the [Markdown Cheat Sheet](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet) will be handy.
- Create a repository with your personal GitHub account, and upload your module installer to the repo
- Create a request for submission via this link: [submission](https://github.com/Magisk-Modules-Repo/submission)

## Module Tricks

#### Remove Files
How to remove a file systemless-ly? To actually make the file *disappear* is complicated (possible, not worth the effort). Replacing it with a dummy file should be good enough! Create an empty file with the same name and place it in the same path within a module, it shall replace your target file with a dummy file.

#### Remove Folders
Same as mentioned above, actually making the folder *disappear* is not worth the effort. Replacing it with an empty folder should be good enough! A handy trick for module developers is to add the folder you want to remove into the `REPLACE` list within `customize.sh`. If your module doesn't provide a corresponding folder, it will create an empty folder, and automatically add `.replace` into the empty folder so the dummy folder will properly replace the one in `/system`.


## Boot Scripts

In Magisk, you can run boot scripts in 2 different modes: **post-fs-data** and **late_start service** mode.

- post-fs-data mode
    - This stage is BLOCKING. The boot process is paused before execution is done, or 10 seconds have passed.
    - Scripts run before any modules are mounted. This allows a module developer to dynamically adjust their modules before it gets mounted.
    - This stage happens before Zygote is started, which pretty much means everything in Android
    - **Run scripts in this mode only if necessary!**
- late_start service mode
    - This stage is NON-BLOCKING. Your script runs in parallel along with the booting process.
    - **This is the recommended stage to run most scripts!**

In Magisk, there are also 2 kinds of scripts: **general scripts** and **module scripts**.

- General Scripts
    - Placed in `/data/adb/post-fs-data.d` or `/data/adb/service.d`
    - Only executed if the script is executable (execution permissions, `chmod +x script.sh`)
    - Scripts in `post-fs-data.d` runs in post-fs-data mode, and scripts in `service.d` runs in late_start service mode.
    - Modules should **NOT** add general scripts since it violates encapsulation
- Module Scripts
    - Placed in the folder of the module
    - Only executed if the module is enabled
    - `post-fs-data.sh` runs in post-fs-data mode, and `service.sh` runs in late_start service mode.
    - Modules require boot scripts should **ONLY** use module scripts instead of general scripts

These scripts will run in Magisk's BusyBox `ash` shell with "Standalone Mode" enabled.

## Root Directory Overlay System

Since `/` is read-only on system-as-root devices, Magisk provides an overlay system to enable developers to replace files in rootdir or add new `*.rc` scripts. This feature is designed mostly for custom kernel developers.

Overlay files shall be placed in the `overlay.d` folder in boot image ramdisk, and they follow these rules:

1. All `*.rc` files in `overlay.d` will be read and concatenated **AFTER** `init.rc`
2. Existing files can be replaced by files located at the same relative path
3. Files that correspond to a non-existing file will be ignored

In order to have additional files that you want to reference in your custom `*.rc` scripts, add them in `overlay.d/sbin`. The 3 rules above does not apply to everything in this specific folder, as they will directly be copied to Magisk's internal `tmpfs` directory (which used to always be located at `/sbin`).

Due to changes in Android 11, the `/sbin` folder is no longer guaranteed to exist. In that case, Magisk randomly generates the `tmpfs` folder. Every occurrence of the pattern `${MAGISKTMP}` in your `*.rc` scripts will be replaced with the Magisk `tmpfs` folder when `magiskinit` injects it into `init.rc`. This also works on pre Android 11 devices as `${MAGISKTMP}` will simply be replaced with `/sbin` in this case, so the best practice is to **NEVER** hardcode `/sbin` in your `*.rc` scripts when referencing additional files.

Here is an example of how to setup `overlay.d` with custom `*.rc` script:

```
ramdisk
│
├── overlay.d
│   ├── sbin
│   │   ├── libfoo.ko      <--- These 2 files will be copied
│   │   └── myscript.sh    <--- to Magisk's tmpfs directory
│   ├── custom.rc          <--- This file will be injected into init.rc
│   ├── res
│   │   └── random.png     <--- This file will replace /res/random.png
│   └── new_file           <--- This file will be ignored because
│                               /new_file does not exist
├── res
│   └── random.png         <--- This file will be replaced by
│                               /overlay.d/res/random.png
├── ...
├── ...  /* The rest of initramfs files */
│
```

Here is an example of the `custom.rc`:

```
# Use ${MAGISKTMP} to refer to Magisk's tmpfs directory

on early-init
    setprop sys.example.foo bar
    insmod ${MAGISKTMP}/libfoo.ko
    start myservice

service myservice ${MAGISKTMP}/myscript.sh
    oneshot
```
