---
title: 安卓APP实现飞行模式自动控制
date: 2025-09-03T15:05:40
slug: appairplane
tags:
  - "#Android"
categories:
  - "#技术探究"
description: 对飞行模式自动控制的方式研究
summary: 简述了飞行模式控制策略，和自动化飞行模式切换的实现方式
cover:
 image:
draft: false
share: true
---
## 前言

本文主要记录 在安卓设备上进行 自动控制切换飞行模式状态 所遇到的问题和解决方案。
本文代码实现在：
https://github.com/powerli2002/AirplaneControl

## 飞行模式控制策略

###  `WRITE _SECURE_ SETTINGS` 权限控制

最王道的方法，通过 adb 授权 应用得到 ` WRITE _SECURE_ SETTINGS` 权限后，可以直接通过参数控制飞行模式。具体控制参数为 `Settings.Global.AIRPLANE_MODE_ON`
授权命令参考：
```shell
adb shell pm grant com.example.xxx android.permission.WRITE_SECURE_SETTINGS
```

代码示例如下：
```java
boolean ok = Settings.Global.putInt(resolver, Settings.Global.AIRPLANE_MODE_ON, enable ? 1 : 0);

```

此方法在 android atudio 的虚拟机 (API35, 安卓 14) 上表现良好，但是在 真实机器上无法执行，表现为，设置飞行模式参数后，只有 蓝牙 变化，而 飞行模式 和 WIFI，移动数据 开关均无反应。
使用 adb shell 直接控制飞行模式开关，表现正常，即直接开关了飞行模式。

当 APP 使用该参数 开启飞行模式时，使用 adb 查看参数状态
```java
➜  ~ adb -s 6d5e7c6c shell settings list global | grep airplane
airplane_mode_on=1
airplane_mode_radios=cell,bluetooth,uwb,wifi,wimax
airplane_mode_toggleable_radios=bluetooth,wifi
bluetooth_airplane_toast_count=10

```

实际上该参数被成功控制。猜测原因可能为，系统对于 adb 和普通 APP 的权限控制不一致，对于普通 APP 的敏感参数控制行为，可能会进行过滤。

### 数字助理权限控制
使用数字助理权限控制进行绕过，参考了 Tasker 的飞行模式控制实现方式。 具体实现方式为：自行创建一个简易的“数字助理”，当程序调用 `showAssist` 方法时，调用该助理，以数字助理的身份进行飞行模式的切换。

#### 核心代码实现

1. **`AndroidManifest.xml` 中声明服务** 需要声明 `VoiceInteractionService` 并指定 Session 类，同时请求 `BIND_VOICE_INTERACTION` 权限。

```xml
<manifest ...>
	<uses-permission android:name="android.permission.BIND_VOICE_INTERACTION" />
	<application ...>
		...
		<service
			android:name=".services.MyInteractionService"
			android:label="IP Geo Collector 助理"
			android:permission="android.permission.BIND_VOICE_INTERACTION"
			android:exported="true">
			<meta-data
				android:name="android.voice_interaction"
				android:resource="@xml/interaction_service" />
			<intent-filter>
				<action android:name="android.service.voice.VoiceInteractionService" />
			</intent-filter>
		</service>
		...
	</application>
</manifest>
```

还需要在 `res/xml/` 目录下创建一个 `interaction_service.xml` 文件，用于指定 Session 类和设置：
```xml
<voice-interaction-service xmlns:android="http://schemas.android.com/apk/res/android"
	android:sessionService="com.example.ipgeocollector.services.MyInteractionSessionService"
	android:recognitionService="none"
	android:supportsAssist="true" />
```

2. **实现 `VoiceInteractionSessionService` 和 `VoiceInteractionSession`** `MyInteractionSessionService` 只是一个返回 `MyInteractionSession` 的简单 Service。核心逻辑在 `MyInteractionSession` 中。

```java
// MyInteractionSession.java
public class MyInteractionSession extends VoiceInteractionSession {

	public MyInteractionSession(Context context) {
		super(context);
	}

	@Override
	public void onHandleAssist(Bundle data, AssistStructure structure, AssistContent content) {
		super.onHandleAssist(data, structure, content);
		// 从 showAssist(args) 的 Bundle 中解析命令
		String command = data.getString("command");

		if ("smart_toggle".equals(command)) {
			// 在这里执行飞行模式切换的逻辑
			executeAirplaneModeToggle();
		}
		// 操作完成后结束 Session
		finish();
	}

	private void executeAirplaneModeToggle() {
		// 通过发送 Intent.ACTION_REQUEST_SET_AIRPLANE_MODE 来请求系统UI进行切换
		// 这是数字助理可以执行的特权操作
		Intent intent = new Intent(Intent.ACTION_REQUEST_SET_AIRPLANE_MODE);
		intent.putExtra("state", !isAirplaneModeOn()); // 切换到相反的状态
		intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
		try {
			startVoiceActivity(intent);
		} catch (Exception e) {
			// 处理异常，如 session hidden
		}
	}

	private boolean isAirplaneModeOn() {
		return Settings.Global.getInt(getContext().getContentResolver(),
									   Settings.Global.AIRPLANE_MODE_ON, 0) != 0;
	}
}
```

## 自动化实现
采用迂回策略：从后台服务中启动一个透明 Activity，利用这个 Activity 作为调用 `showAssist` 的“跳板”。
该自动化实现方式仍然存在缺陷：无法熄屏执行，无法后台静默执行。 原因是 `showAssist` 方法必须使 Activity 保持前台状态并拥有窗口焦点。
### 在透明 Activity 启动 showAssist 的时机

直接在 `onCreate` 或 `onResume` 中调用 `showAssist` 会失败。原因是 Activity 的生命周期回调（`onResume`）和其窗口（Window）真正获得系统焦点之间存在一个微小的时间差。在 `onResume` 执行的瞬间，窗口尚未完全准备好，导致 `showAssist` 请求被系统静默忽略。
正确的姿势是利用 `onWindowFocusChanged` 回调。这个回调是窗口管理系统（Window Manager）发出的明确信号，保证了在执行 `showAssist` 时，我们的 Activity 窗口已经获得了输入焦点，从而满足了 `showAssist` 的先决条件。
```Java
public class TransparentActivity extends Activity {
    private boolean assistTriggered = false;
    @Override
    public void onWindowFocusChanged(boolean hasFocus) {
        super.onWindowFocusChanged(hasFocus);
        if (hasFocus && !assistTriggered) {
            assistTriggered = true;
            try {
                Bundle args = new Bundle();
                args.putString("command", "smart_toggle");
                showAssist(args);
                new Handler(Looper.getMainLooper()).postDelayed(this::finish, 500);
            } catch (Exception e) {
                finish();
            }
        }
    }
}
```

### 优化：避免返回主界面
一个常见的副作用是，当 `TransparentActivity` 执行完任务并 `finish()` 后，系统会返回到任务栈的上一层，也就是我们的主界面 `AirplaneModeActivity`，看起来就像“每次切换都会拉起主界面”。
为了实现真正的静默执行，我们需要让 `TransparentActivity` 在一个完全独立的任务栈中运行，并在结束后彻底销毁该任务栈。
1. **修改 `AndroidManifest.xml`** 为 `TransparentActivity` 增加 `taskAffinity`、`noHistory` 和 `excludeFromRecents` 属性。
- `taskAffinity`: 为 Activity 指定一个独立的任务栈。
- `noHistory`: 该 Activity 不会保留在历史堆栈中。
- `excludeFromRecents`: 该任务不会出现在“最近任务”列表中。

```xml
<activity
	android:name=".ui.airplanemode.TransparentActivity"
	android:theme="@android:style/Theme.Translucent.NoTitleBar"
	android:taskAffinity="com.example.ipgeocollector.assist_task"
	android:noHistory="true"
	android:excludeFromRecents="true"
	/>
```

2. **修改启动方式** 从 `Service` 中启动 `TransparentActivity` 时，添加 `FLAG_ACTIVITY_NEW_TASK` 和 `FLAG_ACTIVITY_MULTIPLE_TASK` 标志。

- `FLAG_ACTIVITY_NEW_TASK`: 与 `taskAffinity` 配合，在新任务栈中启动 Activity。
- `FLAG_ACTIVITY_MULTIPLE_TASK`: 确保每次都能创建一个新任务，即使已有同类任务存在。

```java
Intent intent = new Intent(this, TransparentActivity.class);
intent.addFlags(
	Intent.FLAG_ACTIVITY_NEW_TASK | 
	Intent.FLAG_ACTIVITY_MULTIPLE_TASK | 
	Intent.FLAG_ACTIVITY_NO_ANIMATION
);
startActivity(intent);
```

3. **修改结束方式** 使用 `finishAndRemoveTask()` (API 21+) 来结束 Activity。这个方法会同时销毁 Activity 和它所在的整个任务栈，从而避免了返回上一个界面的问题。

```java
// 在 TransparentActivity 的 onWindowFocusChanged 中
new Handler(Looper.getMainLooper()).postDelayed(() -> {
	if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
		finishAndRemoveTask();
	} else {
		finish();
	}
}, 500);
```

## Reference

- https://stackoverflow.com/questions/77299222/turn-on-off-airplane-mode-in-android-using-device-assistant/78492886#78492886
- https://www.reddit.com/r/tasker/comments/tt5rv5/dev_tasker_606beta_airplane_mode_without_root_or/
- https://github.com/powerli2002/AirplaneControl
- https://developer.android.com/training/articles/assistant?hl=zh-cn
- https://android.googlesource.com/platform/frameworks/base/+/marshmallow-release/tests/VoiceInteraction/
