---
title: XCTrace实现iOS启动时间自动化
date: 2021-04-06 20:17:47
tags: 工作
categories: 工作
---

# 有关启动时间
**启动方式**
- 冷启动：当应用启动时，后台没有该应用的进程，这时系统会重新创建一个新的进程分配给该应用， 这种启动方式就叫做冷启动。（即后台不存在该应用进程）
- 热启动：当应用已经被启动后， 后续按下返回键、Home键等回到桌面或者跳转至其他程序时，二次打开该app的方式叫做热启动。（即后台已经存在该应用进程，系统要做的就是将活动再次置于前台） 

>  参考链接--[https://developer.android.com/topic/performance/vitals/launch-time#cold](https://developer.android.com/topic/performance/vitals/launch-time#cold)

**启动时间**
- 当前「iOS启动时间」计算的是冷启动时间，在APP首次启动以后拿到初始帧渲染（Initial Frame Rendering）完成的时间点，其间APP启动大致分为以下几个过程：

[![pkeCoCD.webp](https://s21.ax1x.com/2024/05/11/pkeCoCD.webp)](https://imgse.com/i/pkeCoCD)


---

# 探究过程
**有关iOS启动时间的探究，初始方向有以下几种：**
- 利用WebDriverAgent工具去实现启动APP的操作，调用系统API去录屏。通过输出视频的前后帧变化来判定是启动完成，从而记录启动时间。
    - 优点：对APP重签名、iphone设备的证书没有限制，且适用范围广。
    - 缺点：过程很繁琐，而且依据前后帧不变做判定不是个稳定的条件，开屏过程中可能出现其他前后帧相同的情况。

- 通过埋点来计算启动时间。记录点击APP-icon埋点的触发时间点、APP首页某个组件埋点曝光结束的时间点，再计算时间差作为启动时间。
    - 优点：启动时间可信度较高
    - 缺点：依赖开发对APP进行埋点

- 通过Instruments命令行工具计算启动时间。利用Instruments命令行调用APP生成.trace文件，通过开源工程[TraceUtility](https://github.com/Qusic/TraceUtility)对.trace文件进行解析。
    - 优点：命令行工具易用性高，且有现有工程可以利用
    - 缺点：输出的.trace文件是二进制格式，只能用可视化Instruments工具查看。想要至为可视化文件，需要TraceUtility工程利用dSYM文件解析对.trace文件进行解析。但TraceUtility实质是个iOS工程，调用了Instruments部分私有API，仅对Xcode 10.0以下版本支持。

- 通过XCTrace命令行工具计算启动时间。
    - 优点： 同是命令行工具，相比较Instruments工具，可输出可视化的文件格式
    - 缺点： 需要UDID在开发者账号下的iphone和开发者证书的包，且开发者账号与开发者证书要一致（即所属budleID要一致）。

**XCTrace探究过程**
- XCTrace生成.trace文件shell命令
`xcrun xctrace record --template 'App Launch' --device  '对应的deciceID' --output ~/Downloads/test.trace --launch XXXX.app'`
该命令中--template参数为想要记录的模版，为了计算启动时间我选择的是'App Launch'。--device参数为Iphone的UDID，当然也可以选择Iphone的设备名称。

- 对.trace文件进行解析，拿到对应的记录结构。
`xcrun xctrace export --input ~/Downloads/test.trace --toc
`
[![pkeC7gH.webp](https://s21.ax1x.com/2024/05/11/pkeC7gH.webp)](https://imgse.com/i/pkeC7gH)

该步骤是为了拿到.trace文件中记录了哪些schema名（可以简单理解成.trace将所有数据以很多张表的形式进行记录，输出的schema就是每张表的名称），作为想要可视化的xpath输入。

- XCTrace解析.trace文件，输出可视化。
`xcrun xctrace export --input ~/Downloads/test.trace --xpath '/trace-toc/run[@number="1"]/data/table[@schema="life-cycle-period"]' >> test.xml`
该步骤的schema即是上一步拿到的模式名，我选取的是'life-cycle-period'。

- XML文件分析及取值。
先用Instruments桌面端工具将记录的.trace文件打开。
[![pkeCT8e.webp](https://s21.ax1x.com/2024/05/11/pkeCT8e.webp)](https://imgse.com/i/pkeCT8e)

观察发现在'Initial Frame Rendering'和'Running in the foreground'进程间有段空置时间，因此不能拿取'Running in the foreground'的开始时间，而是拿取'Initial Frame Rendering'的完成时间。
[![pkeC54O.webp](https://s21.ax1x.com/2024/05/11/pkeC54O.webp)](https://imgse.com/i/pkeC54O)
只需要将该进程的开始时间'start-time'及'duration'相加即可。

---
# 待改进点
1. 由于苹果爸爸的权限管控，目前只能运用于未重签名的包

2. 当前只是冷启动的时间，热启动时间还未拿到。