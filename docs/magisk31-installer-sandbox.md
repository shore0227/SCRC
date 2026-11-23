# Magisk 31 安装器沙箱导致 `cosa database 缺失` 误报

## 现象

在 Magisk 31.1 + Android 16（ColorOS 16）上，从 Magisk App 刷入 SCRC v5.8 时，安装日志固定停在：

```text
云控配置注入安装开始 ...
正在检测 cosa database ... 缺失
请进入一次任意游戏后再次尝试
请确保已启用游戏助手和应用增强服务
Error: 安装失败
```

但同一台设备上：

- `su -c '[ -f /data/data/com.oplus.cosa/databases/db_game_database ] && echo yes'` → `yes`
- `ls -lZ /data/data/com.oplus.cosa/databases/` → 文件存在，`660 u0_aXXX:u0_aXXX`
- `getenforce` → `Enforcing`，`dmesg | grep avc` 无相关 denial

也就是说，**数据库本身存在且可访问**，只是安装器进程看不到它。

## 根因

从 Magisk App 发起安装时，`customize.sh` 继承的 mount namespace 中，`/data/data` 和 `/data/user` 被替换为空 tmpfs（应用数据隔离），只包含 Magisk App 自己的数据：

```text
14802 13809 0:322 / /data/data rw,nosuid,nodev,noexec,relatime - tmpfs tmpfs rw,seclabel,mode=751
14803 13809 0:323 / /data/user rw,nosuid,nodev,noexec,relatime - tmpfs tmpfs rw,seclabel,mode=751
/data/user/0 -> /data/data
```

因此 `scrc_setup.sh` 中的：

```sh
DB_PATH1="/data/data/com.oplus.cosa/databases/db_game_database"
[ -f "$DB_PATH1" ]
```

在该命名空间中必然为假。安装进程本身是 `uid=0`、`CapEff=000001ffffffffff`、`u:r:magisk:s0`，权限与 SELinux 均正常；`pm path com.oplus.cosa` 也仍能返回 APK，因为 PackageManager 走 binder，不依赖文件系统。

而开机时 `service.sh → inject.sh → bin/inject` 运行在正常命名空间，`/data/data` 可见，注入本身不受影响。

## 修复

安装阶段检测到“`/data/data` 是 tmpfs 挂载”时，视为安装器沙箱，跳过 DB 存在性检查并继续安装；正常环境下仍保留原来的报错逻辑：

```sh
DB_SANDBOX=0
if [ "$DB_FOUND" -ne 1 ] && /system/bin/grep -q ' /data/data .* - tmpfs ' /proc/self/mountinfo 2>/dev/null; then
  DB_SANDBOX=1
  DB_FOUND=1
  DB_PATH="$DB_PATH1"
fi
```

补丁文件：`patches/magisk31-installer-sandbox.patch`，作用于分发包中的 `script/scrc_setup.sh`（v5.8）：

```sh
git apply patches/magisk31-installer-sandbox.patch
# 或
patch -p1 < patches/magisk31-installer-sandbox.patch
```

## 验证

测试环境：OnePlus SM8650 设备，ColorOS 16 / Android 16，Magisk 31.1，SCRC v5.8。

安装日志：

```text
正在检测 scx/hmbird 调频器 ...存在
---------------------------------------
云控配置注入安装开始 ...
正在检测 cosa database ... 安装器沙箱内不可见（开机后由 inject 自动检测）
正在尝试恢复官调组件 ...
...
Success: 安装成功
---------------------------------------
```

刷入后 `action.sh`：

```text
正在检测 scx/hmbird 调频器... 存在
检测 persist.sys.oiface.enable ... 已启用
检测 persist.sys.oplus.gameswitch.enable ... 已启用
检测 oiface/gameopt/urcc... 正在运行
执行注入 ...
[成功] [更新] com.kurogame.mingchao
```

注入后的 `db_game_database`：

- `sqlite_master` 中出现 3 个 `protect_local_pkg_*` 触发器
- `PackageConfigBean` 中对应游戏 `from_server=0`，配置列（`cpu_config`、`game_config`、`gpa_config`、`game_zone`、`thermal_frame`、`fps_stabilizer` 等）为合法 JSON
- `PRAGMA integrity_check` → `ok`

## 备注

- 该 sandbox 是 Magisk 31.1 安装器的环境行为，不是数据库或模块脚本内容错误；旧版 Magisk 通常不会触发。
- 如果上游源码不在本仓库，请把本补丁应用到实际发布源码的 `script/scrc_setup.sh` 即可；其余逻辑无需改动。
- 临时绕过：在 root 终端执行 `magisk --install-module <zip>`，root shell 的命名空间正常，可完成安装。