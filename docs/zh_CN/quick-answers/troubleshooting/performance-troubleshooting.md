---
layout: default
title: 性能疑难解答
pagename: performance-troubleshooting
lang: zh_CN
---

# 性能疑难解答

您可能会遇到 Toolkit 使用速度变慢的情况。遇到这种情况的原因有很多，从客户端基础设施问题（例如服务器速度或 Internet 连接），到基于配置的问题（Toolkit 或 Shotgun 没有以高性能的方式配置），再到可用于进一步优化的代码领域。

以下是所要检查事项的快速列表，下面我们将详细介绍这些内容：

- 确保您的应用、插件、框架、核心和 Shotgun Desktop [都是最新的](#keeping-up-to-date)。
- 确保在常规使用期间没有启用[调试日志记录](./turn-debug-logging-on.md)。
- 仅[创建所需的文件夹](#folder-creation-is-slow)并限制文件夹，以便仅在确实需要这些文件夹时才创建它们。在数据结构中添加太多文件夹会降低速度。
- 将用户缓存存储在服务器上可能会很慢。通过将 [`SHOTGUN_HOME` 环境变量](https://developer.shotgunsoftware.com/tk-core/initializing.html#environment-variables)设置为指向本地驱动器上的位置，来重定向用户的 Shotgun 缓存。
- [配置 Workfiles 和加载器应用](#file-open-file-save-or-the-loader-app-is-slow)以过滤出美工人员不需要的内容。考虑按状态进行过滤，这有助于保持实体列表简短且与美工人员的当前任务相关。
- 检查您是否有任何自定义挂钩，并且它们不会增加额外开销。

下面列出了一些良好做法和常见的性能下降场景。这不是一个详尽的列表，当我们看到新的模式时，可以尝试将它添加到列表中。如果本手册不能帮助您找到您所面临问题的根源，请随时提交[支持工单](https://support.shotgunsoftware.com/hc/en-us/requests/new)，我们的团队将很乐意进一步帮助您。

目录：
- [常规良好做法](#general-good-practice)
   - [缓存位置](#cache-location)
   - [保持更新](#keeping-up-to-date)
   - [集中式配置与分布式配置](#centralized-configs-vs-distributed-configs)
   - [调试](#debugging)
- [启动软件时速度很慢](#launching-software-is-slow)
   - [诊断](#diagnosis)
   - [问题是在启动前还是启动后发生？](#is-the-issue-pre-or-post-launch)
   - [检查日志](#checking-the-logs)
   - [软件启动速度慢的常见原因](#common-causes-of-slow-software-launches)
- [“File Open”、“File Save”或加载器应用很慢？](#file-open-file-save-or-the-loader-app-is-slow)
- [文件夹创建速度很慢](#folder-creation-is-slow)
   - [解决 I/O 使用问题](#tackling-io-usage)
   - [注册文件夹](#registering-folders)

## 常规良好做法

### 缓存位置

Shotgun Toolkit [将数据缓存到用户的主目录](../administering/where-is-my-cache.md)。此缓存可以包括许多不同的 SQLite 数据库以及缓存的应用和配置。通常，用户的主目录存储在计算机的本地硬盘驱动器上，但是工作室将它们重定向到网络存储是相当常见的。这样做会影响性能，尤其是 SQLite 数据库，这些数据库用于浏览器集成和文件夹创建/查找等。

如果您的用户目录存储在服务器位置，我们建议使用 [`SHOTGUN_HOME` 环境变量](https://developer.shotgunsoftware.com/tk-core/initializing.html#environment-variables)重新指定 Shotgun Toolkit 缓存的路径。`SHOTGUN_HOME` 环境变量用于设置 Toolkit 缓存各种数据的位置，例如包缓存、缩略图、用于快速查找数据和其他内容的 SQLite 数据库。

### 调试

您可以在 Shotgun Toolkit 中启用调试日志记录，以便能够从各个进程获取更详细的输出。这在尝试诊断问题时非常有用，但是，调试设置不是设计为在日常使用期间启用的。日志记录输出的增加可能会对性能产生显著影响。

当遇到性能问题时，尤其是本地化到特定计算机或用户的问题时，请先确认未启用[调试日志记录](./turn-debug-logging-on.md)。

### 保持更新

如果您遇到性能问题，请检查您的核心、应用、插件和框架是否最新，因为较新版本中可能已经进行了修复或优化。

### 集中式配置与分布式配置

可以采用两种不同的方法设置高级 Toolkit 配置：[集中式和分布式](https://developer.shotgunsoftware.com/tk-core/initializing.html#the-toolkit-startup)。主要区别是，集中式配置通常位于工作室的网络存储中，所有用户都可以访问这些配置，分布式配置通常存储在相关服务中，并按用户在本地缓存。

虽然这两种方法之间的差异超出了性能范围，但它们都会在性能方面带来一些优点和缺点。下表仅从性能角度展示了利弊。

|  | 优点 | 缺点 |
|-------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **集中式配置** | - 一旦初始设置过程完成，它所需的一切内容均已下载，可供所有用户使用。 | - 集中式配置通常保存在网络存储中，因此在正常的 Toolkit 使用期间性能可能会降低。 |
|  | - 未来的更新只需要在集中位置下载一次。 | - Toolkit 配置包含许多小文件，处理大量小文件的元数据操作对服务器而言可能要慢得多，也更难。此外，通过使用 Toolkit 或通过常规使用服务器进行大量读取操作可能会因为无法快速读取配置而影响 Toolkit 的性能。 |
| **分布式配置** | - 缓存的应用、插件、框架和核心采用一种可以与其他本地缓存的配置共享的方式存储。这意味着，如果不同的项目共享相同的依赖项，则这些项目的后续加载可能会更快地缓存。 | - 分布式配置需要按用户在本地缓存。通常，这包括下载配置和所有必需的应用、插件、框架和核心。 |
|  | - 它们存储在用户本地硬盘驱动器上的缓存中，因此通常比服务器速度更快。这意味着，在初始缓存后，性能应该优于集中式配置。 | - 这一过程会在后台无缝进行，但是仍然需要下载这些文件的初始成本。 |
|  |  | - 每次更新配置以指向依赖项的新版本时，都需要缓存配置和新的依赖项。 |

总之，如果您的存储速度较慢，但是 Internet 连接良好，那么分布式配置可能是最佳的解决方案，但是如果您的服务器存储性能很好，而 Internet 连接很差，那么集中式配置可能更合适。

{% include info title="注意" content="如果您对分布式配置感兴趣，但担心按计算机下载依赖项，则可以仅集中包缓存，以便在所有用户之间共享。" %}

当使用分布式配置时，用户只需在缓存中找不到相关内容时才下载它，一旦某个用户已下载它，其他用户也能够利用它。为此，您可以在每台计算机上设置 [`SHOTGUN_BUNDLE_CACHE_PATH` 环境变量](https://developer.shotgunsoftware.com/tk-core/initializing.html#environment-variables) 以指向共享位置。

## 启动软件时速度很慢

您可能会注意到，当启动诸如 Maya、Nuke、Houdini 或其他软件时，启动时间比不带 Shotgun 时要长。这是正常的，它们可能比不带 Shotgun 时稍长片刻，但有时这些时间可能会增加到难以接受的水平（通常取决于我们期望它们在一分钟内启动的软件）。这可能是诊断起来比较棘手的的领域之一，因为启动软件涉及许多过程。

### 诊断
首先要做的是弄清楚这是在什么条件下发生的。

1. **不带 Shotgun 时启动速度慢吗？** - 这似乎是显而易见的，但应注意检查是否仅在带有 Shotgun 的情况下启动软件时才发生该问题。
2. **不管您使用哪种方法启动，速度都很慢吗？也就是说，如果您从 SG Desktop 启动或使用浏览器集成从 SG 站点启动，情况是否大致相同？** - 如果是从 Shotgun 站点而不是从 SG Desktop 启动很慢，那么这可能是浏览器集成的问题，或者它可能意味着在磁盘上创建文件夹的问题。如果是从项目以外的上下文启动，那么很可能是在磁盘上创建了更多文件夹，所以这可能解释了所花的时间。同样值得注意的是，每次启动软件时，我们都会检查所需的文件夹是否存在。
3. **是否在所有项目中都发生？** - 如果不是，那么它很可能特定于配置的设置方式。
4. **是否发生在一天中的特定时刻？** - 如果是，那么这可能表明对基础设施的需求较高，例如在一天的某些时间服务器使用率较高。
5. **是否针对使用的所有计算机/操作系统发生此情况？** - 如果特定计算机速度很慢，则可能是 Toolkit 以外的某些内容导致了问题。但是，清除该计算机上的 Toolkit 缓存是一个良好的开端。不同的操作系统随附不同版本的软件和 Python 软件包，有时可能在特定的内部版本中出现性能问题。具体来说，我们已经看到在 Windows 上使用 Samba (SMB) 共享的性能问题。目前还没有解决此问题的方法，但是如果您正在使用它，那么最好能够意识到这一点。如果您认为该问题仅限于特定的操作系统、Python 软件包或软件版本，请联系我们的[支持团队](https://support.shotgunsoftware.com/hc/en-us/requests/new)，以便他们可以进一步调查。
6. **是否针对所有用户都发生这种情况？** - 与上文类似，如果是同一台计算机上的其他用户，这个问题可能会消失。在这种情况下，首先清除用户的本地 Shotgun 缓存。此外，请确保没有为正常的生产用途启用调试日志记录，因为这将影响性能。
7. **启动速度慢局限于特定的应用/软件，还是所有应用/软件的启动速度都异常缓慢？** - 如果特定软件启动缓慢，这可能意味着存在配置问题。可能有必要检查一下，是否有任何自定义挂钩设置为在启动之前或之后运行，这可能会影响性能。启动时使用的常见挂钩是 [`before_app_launch.py`](https://github.com/shotgunsoftware/tk-multi-launchapp/blob/master/hooks/before_app_launch.py)、[`app_launch.py`](https://github.com/shotgunsoftware/tk-multi-launchapp/blob/master/hooks/app_launch.py) 和核心挂钩 [`engine_init.py`](https://github.com/shotgunsoftware/tk-core/blob/master/hooks/engine_init.py)。有时也会出现以下情况：发布了更新版本的软件，而我们的集成启动速度突然慢很多。在这种情况下，您应该联系[支持](https://support.shotgunsoftware.com/hc/en-us/requests/new) 以确认他们是否了解这一点，以及是否有任何已知的修复。请提供您使用的软件版本号（如果适用，包括修补程序/Service Pack），以及您正在运行的 TK 插件和核心的版本。

### 问题是在启动前还是启动后发生？

如果上述内容没有帮助您缩小问题的范围，那么下一步就是确定在启动过程中什么地方出现了问题。当通过 Toolkit 启动软件时，通常可以将其归结为两步过程。

第一步执行一些初始操作，例如收集启动软件所需的信息，从上下文中自动创建文件夹，然后实际启动软件。一旦软件启动，该过程的第二步就会启动 Toolkit 集成。

通常，不需要查看日志，就可以看到性能问题是在该过程的第一步还是第二步：

- 观察软件启动屏幕的启动是否需要很长时间。如果确实如此，则问题可能就在第一步。
- 看到开始时软件相对较快地启动，但随后变得很慢（在完成初始化并出现 Shotgun 菜单后）。如果是这样，则问题属于第二步。

了解这一点将在接下来的查看日志中对您有所帮助。

### 检查日志

现在，您应该知道问题是在启动过程的第一步还是第二步；这将帮助您锁定要查看的日志。日志按插件进行分解，因此，如果问题出现在启动前阶段，那么您需要查看 `tk-desktop.log` 或 `tk-shotgun.log`，具体取决于您是从 SG Desktop 还是 SG 站点启动。

接下来应该做的是启用调试日志记录。{% include info title="注意" content="如果它已经启用，如[上文所述](#debugging)，这可能会导致运行缓慢，所以您还应该在没有启用它的情况下进行测试" %}
启用调试日志记录后，应清除现有日志，然后重现启动过程。然后，您可以使用日志中的时间戳来查看时间跳转出现在何处。

例如，以下是在文件夹创建期间发生 5 秒时间跳转的几行代码：

    2019-05-01 11:27:56,835 [82801 DEBUG sgtk.core.path_cache] Path cache syncing not necessary - local folders already up to date!
    2019-05-01 11:28:01,847 [82801 INFO sgtk.env.asset.tk-shotgun.tk-shotgun-folders] 1 Asset processed - Processed 66 folders on disk.


一旦定位了时间跳转，日志行就有望让您了解在该阶段发生了什么，例如它是在文件夹创建期间发生的，还是在试图获取 Shotgun 连接时发生的。

不过，阅读日志可能比较麻烦，而且内容并不总是有意义，因此，您可以再次联系[支持](https://support.shotgunsoftware.com/hc/en-us/requests/new)来帮助您完成这一点。

### 软件启动速度慢的常见原因

|-------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Internet 速度慢** | 在需要与 Shotgun 站点连接和通信的 Toolkit 使用的几乎每个方面都会受到 Internet 速度慢的影响。在这种情况下，除了启动软件外，通常还会在其他情况下看到速度问题。但是，如果连接不稳定而不是很慢，那么在启动过程中更有可能遇到性能问题（因为在整个过程中要进行相当多的 Shotgun 通信）。 |
| **服务器访问速度慢** | 这肯定会影响启动时间。如果使用的是[集中式配置](#centralized-configs-vs-distributed-configs)（即，您的配置存储在中央服务器上），则在读取配置文件时可能会有大量的 I/O。最重要的是，如果启动软件，则会触发在启动软件的上下文中创建文件夹。这意味着，它将检查是否创建了文件夹，如果没有，则创建文件夹。 |
| **文件夹创建** | 如上所述，文件夹创建可能是导致速度下降的常见原因。[有关详细信息，请参见下面的文件夹创建性能疑难解答。](#folder-creation-is-slow) |

## “File Open”、“File Save”或加载器应用很慢？

首先要做的是将问题范围缩小到相关应用速度慢的某些方面。

- **启动应用或浏览选项卡是否很慢？**
   - 有可能是该应用当前配置为显示太多信息。可以将“我的任务”(My Tasks)选项卡和其他选项卡配置为从列表中过滤出不需要的实体。例如，您可以过滤出处于特定状态的任务，例如“暂停”(On Hold) (`hld`) 或“最终”(Final) (`fin`)。这不仅提供了性能优势，而且还让美工人员仅看到对他们来说重要的信息。可以过滤[加载器应用](https://support.shotgunsoftware.com/hc/en-us/articles/219033078-Load-Published-Files-#The%20tree%20view)和 Workfiles 应用，但是过滤时 Workfiles 目前没有特定的文档部分，不过过滤器可以作为[层次结构设置](https://support.shotgunsoftware.com/hc/en-us/articles/219033088-Your-Work-Files#Step%20filtering)的一部分应用。
   - “File Open”应用的层次结构也可以配置为延迟加载[子项直到它被展开](https://support.shotgunsoftware.com/hc/en-us/articles/219033088-Your-Work-Files#Deferred%20queries)。现在这是默认的配置设置，但是，如果您有较旧的配置，则可能希望过渡到使用该设置。
   - 确认未启用调试日志记录。这可能会导致大量额外的 I/O，因此会降低速度；这些应用确实包含大量的调试输出。
- **打开、保存或创建新文件时是否很慢？**
   - 检查您是否已接管场景操作或动作挂钩，并确定这些功能是否有任何可能会降低速度的自定义行为。
   - 当创建或保存文件时，Workfiles 将确保在上下文中创建所有必需的文件夹。文件夹创建可能是出现性能[问题](#folder-creation-is-slow)的常见原因。

## 文件夹创建速度很慢

文件夹创建包含许多部分，这可能会导致出现问题时进程变慢。

文件夹创建将：
- 同步本地缓存路径。
- 读取配置的数据结构。
- 生成一个路径列表，这些路径应该在特定的上下文中创建。
- 根据本地存储的路径注册表检查路径。
- 如果尚未注册，请尝试在 SG 站点和本地注册新路径。
- 检查以确定文件夹是否实际存在于磁盘上，无论它们是否已注册，如果不存在，则创建文件夹。

简而言之，文件夹创建可能会占用可能磁盘上的大量 I/O，还需要写入本地数据库并与 SG 站点通信。

### 解决 I/O 使用问题

在处理许多小型读写操作时，您的存储可能很慢或效率很低，因此任何可用于改进基础设施的操作都将有助于加快文件夹创建操作的速度。但是，可以在 Toolkit 配置端执行一些操作，以尝试尽可能地减少负担。

首先要做的是将创建的文件夹限制为对该上下文以及您要在其中工作的环境非常重要的文件夹。例如，如果您正在处理 Maya 中镜头的某个任务，那么理想情况下，您只希望针对特定镜头和软件检查并创建文件夹。

基本上，创建允许您保存和发布工作的最低必需文件夹。

#### 随父文件夹一起创建

有一个 [`create_with_parent` 设置](https://support.shotgunsoftware.com/hc/en-us/articles/219039868-Integrations-File-System-Reference#Create%20With%20Parent%20Folder)，可应用于数据结构文件夹。
将其设置为 True 会导致子文件夹随父文件夹一起创建。您应注意避免这种情况，因为将其设置为 True 会导致检查并创建大量的文件夹。

**示例**

如果您有一个镜头序列/镜头文件夹层次结构，并将您的镜头文件夹设置为随其父镜头序列一起创建，那么每当创建镜头序列文件夹时，它将检查所有关联的镜头并为其创建文件夹。

虽然在某些情况下这可能很方便，但它会导致检查更多的文件夹，并可能同时创建更多的文件夹。在这种情况下，如果您要针对镜头的某个任务在 Workfiles 中创建新文件，则会触发创建镜头的父镜头序列文件夹，进而会创建所有子镜头文件夹，而不仅仅是您正在处理的镜头。

{% include info title="注意" content="工序数据结构文件夹的设置默认为 True。" %}

#### 延迟创建
[`defer_creation` 设置](https://support.shotgunsoftware.com/hc/en-us/articles/219039868-Integrations-File-System-Reference#Workspaces%20and%20Deferred%20Folder%20Creation)允许您将文件夹的创建限制为仅在特定插件运行时才创建文件夹，从而进一步细化何时应创建文件夹。您甚至可以使用自定义名称，然后使用 [sgtk API](https://developer.shotgunsoftware.com/tk-core/core.html?highlight=create_#sgtk.Sgtk.create_filesystem_structure) 触发它们的创建。

**示例**

您可能有一组只能在发布阶段创建的文件夹。在这种情况下，您可以将 custom 设置为 maya_publish 的延迟关键字，然后通过 API 创建使用该关键字作为插件名称的文件夹。数据结构中的文件夹可能如下所示：

    # the type of dynamic content
    type: "static"
    # defer creation and only create this folder when Photoshop starts
    defer_creation: "publish"

然后您将使用如下所示的脚本创建文件夹：

```python
sgtk.create_filesystem_structure(entity["type"], entity["id"], engine="publish")
```

**扩展示例**

考虑进一步延迟文件夹，如果在项目的根目录下有许多非动态文件夹，通常只需要创建一次。例如，默认配置的数据结构根目录下的[“editorial”和“reference”](https://github.com/shotgunsoftware/tk-config-default2/tree/master/core/schema/project)文件夹可能只需要在项目开始时创建一次，但默认情况下，每次创建文件夹时都将检查它们是否存在。

为了限制这一点，可以为它们创建 [yml 文件](https://support.shotgunsoftware.com/hc/en-us/articles/219039868-Integrations-File-System-Reference#Static%20folders)，您可以在其中设置一个延迟关键字，这样只有在特定插件中运行文件夹创建或传递该关键字时才会创建它们。您可以将延迟关键字设置为 `tk-shell`，然后通过 tank 命令（如 `tank folders`）运行文件夹创建。

这意味着，只有在通过 tank 命令运行文件夹创建时才会创建这些文件夹，这是 Toolkit 管理员在第一次设置项目时可以执行的操作。或者，您可以编写一个小型脚本，使用 custom 关键字运行文件夹创建，类似于上面的示例。

### 注册文件夹

在文件夹创建过程中，将[注册](../administering/what-is-path-cache.md)文件夹，以便将来可以使用这些路径来查找上下文。[如前所述](#folder-creation-is-slow)，此过程的一部分需要与 Shotgun 站点进行通信，该站点是存储注册表的中央位置。但是，这些注册表也在本地缓存，以便通过工具更快地进行查找。

#### SQLite 数据库

本地[缓存路径](../administering/what-is-path-cache.md)使用 SQLite 数据库来存储数据。如果数据库存储在网络存储上，那么可能会严重影响数据库的读取和写入性能。

#### 初始同步
在某些情况下，需要从头开始为一个已经注册了许多文件夹的项目生成本地缓存（例如，当新用户加入已在进行中的项目时）。这个过程可能会明显延长，但令人高兴的是，对于该项目，这种情况应该仅发生一次。

后续同步将仅提取本地缓存和站点注册之间的差异。如果用户不经常处理项目，并且在会话之间创建了许多文件夹，那么当所有内容都缓存时，他们可能会明显等待很长时间。

我们在这里看到人们采用的一种方法是，将本地缓存的一个合理最新版本传输到用户的计算机。

{% include info title="注意" content="只有在项目中创建了大量文件夹的情况下，才需要使用此方法。" %}

此更新过程可以通过使用核心挂钩 cache_location.py 自动实现。该挂钩可用于设置缓存的位置，而不是更改位置，您可以使用该挂钩将 path_cache.db 文件的一个版本从中央位置复制到用户的默认位置，从而避免执行代价高昂的完全同步。

然后，可以通过从某人的缓存手动复制，或者使用脚本定期传输它，来定期更新集中存储的缓存路径。

{% include warning title="警告" content="cache_location.py 挂钩可用于设置缓存的位置，但是应该避免针对所有用户将此挂钩设置为指向单一位置，因为当一个或多个进程尝试同时编辑数据库时，这可能会导致数据库锁定。" %}