# Cowian

Many embedded linux systems use a tailored distro and read-only rootfs for robustness and upgrade management. However, it is inconvenient for developers. Many cutting-edge features in popular full-fledged distro, such as Debian, are lacked in tailored ones.

Using Debian in embedded linux system works best with a rw rootfs. It is possible to mount the rootfs in read-only mode, but customizing a Debian to work with ro rootfs is not a trivial job and exhaustive testing is required for each system upgrade.

There are two things that Cowian try to manage: the rootfs and the persistent data (pdata for short).

Both of them are managed in a way similar to git version control. Each time the system boots, Cowian **always** creates a new rootfs and a new copy of persistent data via btrfs snapshot and mounts the new rootfs as / and new pdata as /mnt/persistent. In this way, the system preserves a bunch of bootable combinations of rootfs and pdata.
 
+ the system could be rolled back and forth to a bootable state in arbitrary way.
+ system upgrade and factory recovery is much simpler.
+ extremely easy for developers to debug the rootfs and boot related issues.
+ fail-safe is possible.
+ no need to partition the storage device.

Cowian requires a write-friendly storage device, such as eMMC or SSD.

## Assumption

1. Cowbian uses rw btrfs subvolume as /, guarantees best compatibility with Debian (or other full-fledged distro).
2. During boot, Cowbian discards all changes made on previous boot. This is similar to overlay rootfs with a tmpfs, but consumes much less memory. If a dedicated partition is used with overlay rootfs, cowroot avoids a time consuming clean-up or reformat. Also, the previous change are preserved.
3. The strategy is the System App is responsible for all persistent data. It understands all data format and version. It can check the integrity. It can also rollback to previous version.

## Dependency

1. u-boot btrfs support
2. kernel btrfs support

If your uboot does not support btrfs, it is possible to place u-boot script and copy all boot files to a u-boot friendly partition. But this is not recommended. Using single btrfs volume and single update lock is more robust.

## Layer

```
  System App
  ----------
  Debian
  ----------
  rootfs (rw), pdata (rw)
  ----------
  Cowroot
  ----------
```

Cowroot provides a mechanism for choosing rootfs and pdata snapshots to boot Debian. However, It is the System App's responsibility to tell cowroot what is a successful boot.

## Layout

```
/boot             # for u-boot script
/cowroot
  /vols
    /UUID         # ro or rw vol
    /UUID.snaps   # snapshot of rw
  /data           # persistent data UUID
    /UUID
/tmp
  /vols
  /files
```

## Bootable Log

The system app is responsible for writing a boot log.

A bootable log entry is a pair of uuids, representing a rootfs and a pdata snapshot, respectively. The record tells u-boot and cowian such a combination is bootable.

## boot and try boot

u-boot has no idea on WHY a subvolume is chosen to be the rootfs. It concerns only:

1. whether the new rootfs works?
2. if the new rootfs does NOT work, how to roll back to a previously working one.

so u-boot keeps track of a normal boot and a try boot

In normal boot, it simply boots a bootable volume.

In try boot, it boot a volume and keeps a rollback volume. If try boot fails, it knows how to roll back to a previously working copy, with failure information passing to kernel.

## read-only boot

create a tmp rw volume, and use it as rootfs. it is the system app's responsibility to tell u-boot it succeeds if it is a try boot.

create a tmp rw volume, and use it as rootfs. 

## Upgrade

## Factory Reset

## Fail-Safe

## Handover/Handoff Protocol

基础假设是：

1. uboot知道如何在存储介质上根据identifier找到一个可启动系统载入
2. 系统作为一个整体呈现给uboot，uboot既不知道系统内是否分了多个程序分散执行协议逻辑，也不知道具体业务要求，例如是upgrade，rollback, roll forward，还是factory reset。

在模型上这是一个Handover逻辑，一次系统切换看作是一次三方通讯，即bootloader, current和target；系统的初始镜像是保证可启动的。

```
init    -> bl     : go to target
bl      -> target : are you OK?
## success ##
target  -> bl     : yes, fine.
```

```
init    -> bl     : go to target
bl      -> target : are you OK?
## failure ##
bl      -> init   : bad target 
```

实际的过程不是通过消息传递来实现通讯，而是使用一个共享存储空间，可以称之为shared mailbox，在任何时刻，都只有一方处于运行状态，可以对shared mailbox进行读写；

协议设计从immutable and append-only的方式开始，且假设在语义上任何一方的语义均不同。

通讯从initiator请求开始；如果请求失败，u-boot要告知initiator；如果请求成功，u-boot不必告知initiator；换句话说，是否有应答依赖于结果；应答仅仅通过cmdline是不够的，u-boot无法确认initiator是否获得应答，所以应答必须通过持久化方式传递，相当于initiator确认受到了应答。

```
[initiator] transfer to TARGET
[u-boot]    trying TARGET
[u-boot]    failed
```
If failed, it is the initiator's responsibility to clear all message. It must remove of file one by one in reverse order

```
[]
```

```
[boot]      init
[init]      go to target
[try-boot]      
[boot]      init
```






