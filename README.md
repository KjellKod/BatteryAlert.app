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
