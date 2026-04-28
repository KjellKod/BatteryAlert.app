# Battery Alert for macOS

A small Automator app that polls your battery level and alerts you when it gets low. No third-party tools, no menu bar app, no daemon — just AppleScript and the built-in `pmset` command.

## Setup

1. Open **Automator**.
2. Choose **New Document → Application**.
3. In the actions library, search for **Run AppleScript**.
4. Drag **Run AppleScript** into the workflow area on the right.
5. Replace the placeholder script with the script below.
6. Save it as **Battery Alert.app** in your **Applications** folder.

## The script

```applescript
use scripting additions

on run {input, parameters}
	set earlyWarned to false
	set criticalWarned to false
	
	repeat
		set batteryInfo to do shell script "pmset -g batt"
		set batteryPercentText to do shell script "pmset -g batt | grep -Eo '[0-9]+%' | head -1 | tr -d '%'"
		set batteryPercent to batteryPercentText as integer
		set isDischarging to batteryInfo contains "discharging"
		
		do shell script "echo \"$(date) pct=" & batteryPercent & " discharging=" & isDischarging & "\" > /tmp/battery_alert.log"
		
		if isDischarging then
			if batteryPercent ≤ 10 then
				if criticalWarned is false then
					display alert "Critical battery warning" message ¬
						"Your MacBook is at " & batteryPercent & "%. Plug in now." ¬
						as critical buttons {"OK"} default button "OK"
					set criticalWarned to true
					set earlyWarned to true
				end if
			else if batteryPercent ≤ 20 then
				if earlyWarned is false then
					display alert "Low battery warning" message ¬
						"Your MacBook battery is at " & batteryPercent & "%. Plug in soon." ¬
						giving up after 10
					set earlyWarned to true
				end if
			end if
		else
			set earlyWarned to false
			set criticalWarned to false
		end if
		
		delay 60
	end repeat
	
	return input
end run
```

## Run it automatically at login

1. Open **System Settings**.
2. Go to **General → Login Items**.
3. Click **+** under **Open at Login** and add **Battery Alert.app**.

The first time it tries to show an alert, macOS may ask for notification permission. Allow it.

## How it works

The script runs an infinite loop that polls every 60 seconds:

1. Calls `pmset -g batt` to read the current battery state.
2. Extracts the percentage and whether the Mac is discharging (i.e., not plugged in).
3. Logs one line to `/tmp/battery_alert.log` so you can verify it's running.
4. Decides whether to alert based on two thresholds.

There are two alert levels:

- **20% — Low battery warning.** A non-blocking alert that auto-dismisses after 10 seconds. You'll see it but you don't have to click anything.
- **10% — Critical battery warning.** A modal alert that stays on screen until you click **OK**. You can't miss it.

Each alert fires **once per discharge cycle**. After it fires, an internal flag (`earlyWarned` or `criticalWarned`) is set so the same alert won't repeat on the next poll, even though the battery is still under the threshold.

## When the alerts reset

The flags reset the moment the Mac reports it's no longer discharging — i.e., when you plug in the charger. Both `earlyWarned` and `criticalWarned` go back to `false`.

That means a new discharge cycle starts fresh:

- Unplug at 100% → drain to 19% → low-battery alert fires once.
- Continue draining → 11%: silent (already warned).
- Hit 10% → critical alert fires once.
- Plug in → both flags reset.
- Unplug again later above 20% → cycle starts over and you'll see alerts again when thresholds are crossed.

This means you won't get nagged repeatedly during a single discharge, but you also won't be ignored across multiple unplugs.

## Verifying it's running

In Terminal:

```
tail -F /tmp/battery_alert.log
```

You'll see a new line every 60 seconds with the current percentage and discharge state. If lines stop appearing, the app crashed or was quit.

When you're confident it works, you can remove the `do shell script "echo ..."` line to stop writing to the log.

## Customizing

- **Change the thresholds:** edit `≤ 10` and `≤ 20` in the script.
- **Change the poll interval:** edit `delay 60` (the value is in seconds).
- **Add a sound to the soft alert:** add this line right before `display alert` in the 20% branch:

  ```applescript
  do shell script "afplay /System/Library/Sounds/Sosumi.aiff &> /dev/null &"
  ```

  The trailing `&` makes the sound play asynchronously so the script keeps running.

## Limitations

- **State is in memory only.** If you quit and relaunch the app during a single discharge, you'll get a duplicate alert because the flags reset on launch.
- **No nagging.** If you ignore the critical alert and keep using the laptop, the script won't remind you again until you plug in and unplug.
- **One alert per threshold per cycle.** If you want repeated reminders while critically low, the script needs to be modified to use a timer instead of a flag.


# SECURITY
Review this script, make sure it's not malicous or doing things you don't agree with. 

## Why this is needed 

> **Adware and click fraud disguised as utility apps.** In April 2019,
> Google banned developer DO Global (formerly DU Group), removing 46
> of its apps from the Play Store after a BuzzFeed News investigation
> found they were committing click fraud — clicking ads in the
> background without users knowing.[1] DO Global's apps had over 600
> million combined installs;[2] ES File Explorer was the best-known
> name in the ban.[3]
>
> **Cheetah Mobile ban (February 2020).** Google removed roughly 45
> Cheetah Mobile apps from the Play Store as part of a broader
> ~600-app sweep targeting "disruptive ads" and ad fraud.[4][5]
> Cheetah's portfolio included popular battery, cleaner, and security
> utilities.
>
> **Banking trojan delivered via a fake battery saver.** In January
> 2019, Trend Micro researchers documented an app called
> **BatterySaverMobi** on the Play Store carrying the Anubis Android
> banking trojan.[6][7] It used the device's motion sensor to evade
> sandboxes and emulators (only activating when the device was
> actually being moved by a real user), then harvested banking
> credentials via a keylogger and screenshots.[7] The malicious
> payload was disguised as a system update.[6]
>
> **Spyware-grade permission abuse.** A long pattern across "battery,"
> "cleaner," and "booster" apps: requesting permissions (contacts,
> SMS, location) that have no plausible relationship to battery
> monitoring, then exfiltrating that data to ad networks or unknown
> servers.[8] The mismatch between stated function and requested
> permissions is the giveaway.
>
> **Why this category specifically?** Three reasons: (1) "battery" is
> one of the most-searched utility terms, so it draws clicks;
> (2) users grant broad permissions to anything labeled "system
> utility"; (3) genuine battery monitoring on Android needs almost no
> permissions, so a real one looks suspiciously similar to a fake one
> until you watch its behavior.
>
> **References**
>
> [1] Silverman, C. *Popular Apps In Google's Play Store Are Abusing
> Permissions And Committing Ad Fraud.* BuzzFeed News, April 23,
> 2019.
> https://www.buzzfeednews.com/article/craigsilverman/google-play-store-ad-fraud-du-group-baidu
>
> [2] Bradshaw, K. *Google bans Play Store developer w/ over 600
> million installs due to ad fraud scheme.* 9to5Google, April 29,
> 2019. https://9to5google.com/2019/04/29/google-play-store-ban-do-global/
>
> [3] Ruddock, D. *ES File Manager vanishes from Play Store, possibly
> part of DO Global scandal.* Android Police, April 27, 2019.
> https://www.androidpolice.com/2019/04/27/es-file-manager-vanishes-from-play-store-possibly-part-of-do-global-scandal/
>
> [4] Carlon, K. *Google has removed almost all Cheetah Mobile apps
> from the Play Store.* Android Police, February 27, 2020.
> https://www.androidpolice.com/2020/02/27/cheetah-mobile-apps-disappeared-play-store/
>
> [5] Silverman, C. *Google Has Banned Almost 600 Android Apps For
> Pushing "Disruptive" Ads.* BuzzFeed News, February 20, 2020.
> https://www.buzzfeednews.com/article/craigsilverman/google-bans-android-apps-disruptive-ads
>
> [6] Khandelwal, S. *New Android Malware Apps Use Motion Sensor to
> Evade Detection.* The Hacker News, January 18, 2019.
> https://thehackernews.com/2019/01/android-malware-play-store.html
>
> [7] Waqas. *Malicious apps deploy Anubis banking trojan using motion
> detection.* HackRead, January 18, 2019.
> https://hackread.com/malicious-apps-anubis-banking-trojan-motion-detection/
>
> [8] Lookout Threat Intel. *What You Need To Know About the Banking
> Trojan Anubis.* Lookout.
> https://www.lookout.com/threat-intelligence/article/anubis-targets-hundreds-of-financial-apps
