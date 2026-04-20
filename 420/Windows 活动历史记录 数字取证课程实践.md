Windows 活动历史记录 数字取证课程实践

我来帮你了解Windows活动历史记录的数字取证实践。这是一个涉及Windows系统取证分析的专业话题，让我搜索相关的技术资料和最新信息。
根据搜索结果，我为你整理一份关于 **Windows 活动历史记录（Activity History/Timeline）数字取证** 的课程实践指南。

## Windows 活动历史记录数字取证实践

### 一、核心概念与价值

Windows 10 在 1803 版本引入了 **Timeline（时间线）** 功能，通过 **活动历史记录（Activity History）** 追踪用户最近 30 天内的系统活动 。这对数字取证具有重要价值：

- **用户通常未意识到该功能存在** ，因此保留了大量未被篡改的证据
- 可重建用户行为时间线，即使传统日志已被清除
- 能追溯文件访问、应用程序执行、剪贴板操作等详细活动

---

### 二、关键取证文件位置

**主要数据库文件：**

```markdown
%LocalAppData%\ConnectedDevicesPlatform\<用户配置文件>\ActivitiesCache.db
```
![image](/屏幕截图%202026-04-20%20161033.png)
**相关文件：**

- `ActivitiesCache.db` — SQLite 数据库主文件
- `ActivitiesCache.db-shm` — 共享内存文件
- `ActivitiesCache.db-wal` — 预写日志（可能包含未提交数据）

**剪贴板历史（如启用）：**
```markdown
%LocalAppData%\Microsoft\Windows\Clipboard\
```
![image](/屏幕截图%202026-04-20%20161326.png)
使用win+V查看
![image](/屏幕截图%202026-04-20%20181045.png)
---

### 三、数据库结构分析

`ActivitiesCache.db` 是 **SQLite 格式** 数据库，关键表包括：

| 表名 | 取证价值 |
| --- | --- |
| `Activity` | 核心活动记录，包含应用ID、时间戳、Payload |
| `Activity_PackageID` | 应用程序包标识映射 |
| `ActivityOperation` | 操作详情，含剪贴板数据（ClipboardPayload） |

**关键字段解析：**

- **AppID** — 执行的应用程序（如 `Microsoft Word` ）
- **Payload** — JSON 格式数据，包含文件路径、显示文本等
- **StartTime/EndTime** — 纪元时间戳（需转换）
- **ActivityType** — 类型 10 表示剪贴板操作，类型 16 表示复制/粘贴

---

### 四、取证分析步骤

#### 1\. 现场勘查与提取

```powershell
# 定位数据库文件（在管理员 PowerShell 中）
Get-ChildItem -Path "$env:LOCALAPPDATA\ConnectedDevicesPlatform" -Recurse -Filter "ActivitiesCache.db*"
```

**注意事项：**

- 优先采用 **离线提取** （Live Response 或磁盘镜像），避免数据覆盖
- 数据库默认保留约 **30 天** 数据，具有时效性
- 需停止 `CDPUserSvc_*` 服务后再复制，避免文件锁定

#### 2\. 数据库解析

使用 **DB Browser for SQLite** 或 **Autopsy** 等工具打开数据库。
![image](/屏幕截图%202026-04-20%20161754.png)
**关键 SQL 查询示例：**

```sql
-- 查看最近的应用活动
SELECT 
    AppId,
    ActivityType,
    datetime(StartTime/10000000 - 11644473600, 'unixepoch', 'localtime') as StartTime_Local,
    Payload
FROM Activity
ORDER BY StartTime DESC
LIMIT 50;
```
![image](/屏幕截图%202026-04-20%20162616.png)

```sql
-- 查看 Activity 表的全部 ActivityType 分布
SELECT 
    ActivityType,
    COUNT(*) as 记录数量,
    MIN(StartTime) as 最早时间戳,
    MAX(StartTime) as 最晚时间戳
FROM Activity
GROUP BY ActivityType
ORDER BY ActivityType;
```
![image](/屏幕截图%202026-04-20%20165437.png)
```sql
-- 查看type12、15的原始记录
SELECT 
    AppId,
    ActivityType,
    datetime(StartTime, 'unixepoch', 'localtime') as Time,
    Payload
FROM Activity
WHERE ActivityType IN (12, 15)
ORDER BY StartTime DESC;
```
![image](/屏幕截图%202026-04-20%20170235.png)
发现这是 Windows 凭据和加密子系统的审计痕迹
#### 3\. 时间戳转换

Windows 活动历史记录使用 **纪元时间（Epoch Time）** ，需转换为可读格式：

- 通常基于 **1601-01-01** 的 Windows 文件时间
```sql
-- 查询正确的用户活动时间线
SELECT 
    AppId,
    ActivityType,
    datetime(StartTime, 'unixepoch', 'localtime') as StartTime_Local,
    Payload
FROM Activity
WHERE StartTime > 0
ORDER BY StartTime DESC
LIMIT 50;
```
![image](/屏幕截图%202026-04-20%20165813.png)

#### 4\. 关联分析、找到"用户打开文件"的证据

**当前数据库的问题**

你的 `ActivitiesCache.db` 只有 Type 11/12/15（系统配置、凭据、加密）， **没有 Type 5/6（文件打开、应用聚焦）** 。

### 替代数据源：Windows 跳转列表（Jump Lists）
Jump Lists 记录用户最近打开的文件，是活动历史记录的重要补充。

**文件位置**

```markdown
%AppData%\Microsoft\Windows\Recent\AutomaticDestinations\
%AppData%\Microsoft\Windows\Recent\CustomDestinations\
```
#### 提取步骤

**步骤 1：定位 Jump Lists 文件**

```powershell
# 在 PowerShell 中执行
Get-ChildItem "$env:APPDATA\Microsoft\Windows\Recent\AutomaticDestinations" | Select-Object Name, LastWriteTime
```
![image](/屏幕截图%202026-04-20%20172105.png)
**步骤 2：解析 Jump Lists** Jump Lists 是复合二进制文件，需要专用工具：

| 工具 | 用途 |
| --- | --- |
| **JumpListView** (NirSoft) | 免费，图形界面，直接查看 |
| **JLECmd** (Eric Zimmerman) | 命令行，取证标准工具 |
| **Autopsy** | 综合取证平台自动提取 |

#### 使用 JLECmd 提取（推荐）

```powershell
# 下载 JLECmd 后执行
.\JLECmd.exe -d "C:\Users\Lenovo\AppData\Roaming\Microsoft\Windows\Recent\AutomaticDestinations" --csv .\Auto --csvf "All_AutoDest.csv"
```
![image](/屏幕截图%202026-04-20%20174410.png)
输出 CSV 包含：
- 文件名
- 文件路径
- 最后访问时间
- 创建时间
- 应用标识
![image](/屏幕截图%202026-04-20%20175001.png)
---
### 替代数据源：SRUM 数据库

SRUM（System Resource Usage Monitor）保留 30 天应用使用记录。

#### 文件位置

```markdown
C:\Windows\System32\sru\SRUDB.dat
```

#### 提取步骤

**步骤 1：复制 SRUM 数据库到evidence目录（需要 SYSTEM 权限）**
![image](/屏幕截图%202026-04-20%20180133.png)
**步骤 2：使用 srumECmd 解析**
* 工具会自动创建 E:\迅雷下载\SrumECmd\evidence\results 文件夹
解析完成后，里面会生成多个 CSV 文件，核心的有：
SrumECmd_AppResourceUsage.csv：程序运行记录（时间、时长）
SrumECmd_NetworkConnections.csv：网络连接记录（IP、端口）
SrumECmd_NetworkUsage.csv：流量使用情况
![image](/屏幕截图%202026-04-20%20180624.png)
![image](/屏幕截图%202026-04-20%20180802.png)

---

### 替代数据源：LNK 文件（快捷方式）

Windows 自动创建最近文件的快捷方式。

#### 文件位置

```markdown
%AppData%\Microsoft\Windows\Recent\
```
![image](/屏幕截图%202026-04-20%20171300.png)

---


### 五、注意事项与局限性

1. **隐私设置影响** ：用户可在"设置 → 隐私 → 活动历史记录"中关闭收集，或手动清除数据
2. **云同步干扰** ：如启用 Microsoft 账户同步，活动可能跨设备，需区分本地与云端记录
3. **数据库损坏** ：删除操作可能导致记录变为"灰色"残留，需检查 WAL 文件
4. **剪贴板限制** ：仅当启用"剪贴板历史记录"和"跨设备同步"时才会记录

---



---

## Windows通知中心数字取证实践指南（探索式）

### 阶段一：发现目标

**情境** ：您接到任务，需要分析一台Windows电脑的通知历史，但不知道数据存储在哪里。

**步骤1：寻找可能的存储位置**

基于Windows应用数据存储规范，用户数据通常位于：

```powershell
# 查看AppData\Local下的Microsoft相关目录
Get-ChildItem "$env:LOCALAPPDATA\Microsoft" | Where-Object { $_.Name -match "Notif|Push|Toast|Wpn" }

# 全局搜索可能的文件
Get-ChildItem "$env:LOCALAPPDATA" -Recurse -File -ErrorAction SilentlyContinue | 
    Where-Object { $_.Name -match "wpndatabase|notification|push" } |
    Select-Object FullName, Length, LastWriteTime
```
**预期发现** ：您应该能找到路径：

```markdown
C:\Users\[用户名]\AppData\Local\Microsoft\Windows\Notifications\wpndatabase.db
```
**验证发现** ：
![image](/屏幕截图%202026-04-20%20191323.png)

**关键疑问** ：为什么有3个文件？这是什么数据库格式？

---

### 阶段二：识别数据格式

**情境** ：您发现了一个名为 `wpndatabase.db` 的文件，但不知道用什么工具打开。

**步骤1：选择工具**

| 工具选项 | 适用场景 |
| --- | --- |
| DB Browser for SQLite | 图形界面，适合初学者探索 |
| sqlite3命令行 | 脚本化操作，适合批量处理 |
| Python + sqlite3库 | 自动化分析，适合深度取证 |

**建议** ：首次探索使用 **DB Browser for SQLite** （免费，官网下载）。

---

### 阶段三：首次打开数据库（探索模式）

**情境** ：您用DB Browser打开了 `wpndatabase.db` ，看到一堆表名，但不知道哪个重要。

**步骤1：查看所有表**

在DB Browser的"数据库结构"标签页，您会看到：

- Notification
- NotificationHandler
- HandlerAssets
- HandlerSettings
- Metadata
- WNSPushChannel
- NotificationData
- TimedNotification
- TransientTable
![image](/屏幕截图%202026-04-20%20191503.png)
**思考** ：哪个表可能包含实际通知内容？

**步骤2：逐个浏览表内容**

**先查看Metadata** （通常记录数最少，容易理解）：
![image](/屏幕截图%202026-04-20%20190048.png)

**发现与推理** ：

- `maxCount` 表示最大保留数量 → **通知有数量限制，旧的会被删除**
- `CurrentNotificationId=37937` → **系统累计处理过37,937条通知**
- 但 `toast:maxCount=20` → **目前只保留20条弹窗通知**

**结论** ：这个数据库是 **循环覆盖** 的！大部分历史通知已被删除。

---

### 阶段四：寻找当前保留的通知

**情境** ：您知道只有20条Toast通知被保留，想找到它们。

**步骤1：浏览Notification表**

切换到"浏览数据"标签页，选择表： `Notification`

您会看到字段：

- Id, HandlerId, Type, Payload, ArrivalTime, ExpiryTime...
![image](/屏幕截图%202026-04-20%20190125.png)

**观察到的现象** ：

- `Type` 列有 `toast` 和 `tile` 两种值
- `Payload` 列显示 `<toast...` 或 `<tile...` 开头的XML内容
- `ArrivalTime` 是长数字（如 `1342115560...`）

**疑问** ：这些长数字是什么时间格式？

**步骤2：尝试理解时间戳**

在DB Browser中执行SQL测试：

```sql
-- 尝试Unix时间转换
SELECT datetime(ArrivalTime, 'unixepoch') FROM Notification LIMIT 1;
-- 错误日期 → 排除Unix时间戳
-- 尝试Windows FILETIME转换
SELECT datetime((ArrivalTime / 10000000.0 - 11644473600), 'unixepoch', 'localtime') FROM Notification LIMIT 1;
-- 结果应该是合理日期（如2024-XX-XX）
```
![image](/屏幕截图%202026-04-20%20191829.png)
**发现** ：这是 **Windows FILETIME** 格式（1601年1月1日起的100纳秒间隔）。

---

### 阶段五：识别通知来源应用

**情境** ：您看到 `HandlerId` 列是数字（如133, 719, 723），但不知道对应哪个应用。

**步骤1：查找应用名称映射表**

浏览 `HandlerAssets` 表：
![image](/屏幕截图%202026-04-20%20190026.png)

**发现** ： `HandlerId=317` 是"设置"应用， `HandlerId=608` 是"篡改猴"（Tampermonkey浏览器扩展）。

**疑问** ：为什么只有6条记录？Notification表有更多HandlerId！

**步骤2：查找更完整的应用注册信息**

浏览 `NotificationHandler` 表（349条记录）：
![image](/屏幕截图%202026-04-20%20190210.png)

**发现** ： `RecordId` 似乎与 `HandlerId` 对应！ `PrimaryId` 是应用的完整标识名。



---

### 阶段六：建立完整关联查询

**情境** ：您已经知道表之间的关系，现在想建立完整的通知时间线。

**步骤1：编写基础关联查询**

```sql
SELECT 
    n.Id as 通知ID,
    n.HandlerId,
    ha.AssetValue as 应用名称,
    n.Type as 通知类型,
    datetime((n.ArrivalTime / 10000000.0 - 11644473600), 'unixepoch', 'localtime') as 接收时间,
    n.Payload as 原始内容
FROM Notification n
LEFT JOIN HandlerAssets ha 
    ON n.HandlerId = ha.HandlerId 
    AND ha.AssetKey = 'DisplayName'
ORDER BY n.ArrivalTime DESC;
```
![image](/屏幕截图%202026-04-20%20192613.png)
**遇到的问题** ：部分 `HandlerId` 在 `HandlerAssets` 中没有 `DisplayName` 记录。

**解决方案** ：使用 `NotificationHandler` 表作为后备：

```sql
SELECT 
    n.Id as 通知ID,
    n.HandlerId,
    COALESCE(ha.AssetValue, nh.PrimaryId) as 应用标识,
    n.Type as 通知类型,
    datetime((n.ArrivalTime / 10000000.0 - 11644473600), 'unixepoch', 'localtime') as 接收时间,
    substr(n.Payload, 1, 50) as 内容摘要
FROM Notification n
LEFT JOIN HandlerAssets ha 
    ON n.HandlerId = ha.HandlerId AND ha.AssetKey = 'DisplayName'
LEFT JOIN NotificationHandler nh 
    ON n.HandlerId = nh.RecordId
ORDER BY n.ArrivalTime DESC;
```
![image](/屏幕截图%202026-04-20%20192718.png)
---

### 阶段七：解析通知内容（Payload）

**情境** ： `Payload` 字段包含XML，您想提取人类可读的文本。

**步骤1：查看原始XML**

在DB Browser中双击某行的 `Payload` 单元格，右侧显示：

```xml
<toast>
 <visual>
  <binding template="ToastText02">
   <text id="1">任务完成</text>
   <text id="2">您的任务已完成，可返回 TRAE 查看结果。
</text>
  </binding>
 </visual>
 <audio silent="true"/>
</toast>
```
![image](/屏幕截图%202026-04-20%20193057.png)
**步骤2：提取文本的SQL尝试**

```sql
-- 提取 Payload 前 N 个字符预览
SELECT 
    Id,
    HandlerId,
    Type,
    datetime((ArrivalTime / 10000000.0 - 11644473600), 'unixepoch', 'localtime') as 时间,
    substr(Payload, 1, 200) as 内容预览
FROM Notification
WHERE Type = 'toast';
```
![image](/屏幕截图%202026-04-20%20193317.png)
**注意** ：
优点：不会出错，快速查看内容结构
缺点：包含 XML 标签，不够美观。

---

### 阶段八：分析权限配置（HandlerSettings）

**情境** ：您发现了 `HandlerSettings` 表有8,292条记录，想知道这是什么。

**步骤1：查看单个Handler的配置**

```sql
SELECT * FROM HandlerSettings WHERE HandlerId = 1 LIMIT 20;
```
**观察到的数据** ：
![image](/屏幕截图%202026-04-20%20193445.png)
**推理** ：

- 每个HandlerId有多条记录 → **这是权限配置表**
- `Value` 只有0或1 → **布尔型开关**
- `SettingKey` 有前缀（s:, c:, r:）→ **不同前缀代表不同类别**

**步骤2：统计前缀含义**

```sql
SELECT 
    substr(SettingKey, 1, 1) as 前缀,
    COUNT(*) as 数量
FROM HandlerSettings
GROUP BY substr(SettingKey, 1, 1);
```
![image](/屏幕截图%202026-04-20%20193546.png)
**假设验证** ：

- `s:` 最多 → **Setting（实际设置值）**
- `c:` 次之 → **Capability（应用声明的能力）**
- `r:` 最少 → **Restriction（系统限制）**

---

### 阶段九：发现云端推送通道

**情境** ：您看到 `WNSPushChannel` 表，不知道WNS是什么。

**步骤1：查看内容**

```sql
SELECT * FROM WNSPushChannel LIMIT 5;
```

**观察到的数据** ：
![image](/屏幕截图%202026-04-20%20193637.png)
**推理** ：

- `Uri` 包含 `notify.windows.com` → **微软官方域名**
- `https://` 协议 → **云端服务器**
- `HandlerId` 关联应用 → **应用通过此通道接收云端推送**

**搜索验证** （在浏览器或搜索引擎中）：

> "WNS Windows Push Notification Service"

**确认** ：WNS = Windows Push Notification Service，是微软的云端推送服务。

**取证意义** ：即使应用未运行，也能通过WNS接收通知。通知可能来自云端而非本地生成。

---

### 探索过程中的关键决策树

```markdown
开始：寻找Windows通知数据
  │
  ├─► 找到 wpndatabase.db
  │     │
  │     ├─► 识别为SQLite格式 → 使用DB Browser打开
  │     │
  │     ├─► 发现多个表 → 从Metadata开始（数据最少）
  │     │     │
  │     │     └─► 发现 CurrentNotificationId=37937（累计通知数）
  │     │           发现 maxCount限制（Toast仅保留20条）
  │     │
  │     ├─► 查看Notification表 → 发现HandlerId数字
  │     │     │
  │     │     └─► 关联HandlerAssets → 获取应用名称
  │     │           关联NotificationHandler → 获取系统标识
  │     │
  │     ├─► 发现ArrivalTime是长数字 → 测试转换
  │     │     │
  │     │     ├─► Unix时间转换失败（日期错误）
  │     │     │
  │     │     └─► FILETIME转换成功（日期合理）
  │     │
  │     ├─► 查看Payload是XML → 提取文本内容
  │     │
  │     ├─► 发现HandlerSettings表 → 分析权限配置
  │     │     │
  │     │     └─► 识别前缀含义（s:/c:/r:）
  │     │
  │     └─► 发现WNSPushChannel表 → 识别云端推送
  │           │
  │           └─► 验证WNS = Windows Push Notification Service
  │
  └─► 未找到 → 检查Windows版本（Win7/8无此数据库）
        检查用户权限（是否访问了正确用户目录）
```

---

### 您现在所处的阶段
您已完成：
- ✅ 阶段一：发现数据库位置
- ✅ 阶段二：识别SQLite格式
- ✅ 阶段三：打开并浏览表结构
- ✅ 阶段四：发现FILETIME时间戳
- ✅ 阶段五：识别HandlerId映射关系
- ✅ 阶段六：建立关联查询
- ✅ 阶段七：查看Payload XML内容
- ✅ 阶段八：浏览HandlerSettings
- ✅ 阶段九：发现WNSPushChannel
