# Resetting Passwords
## (badpassword)
- Catalyst 3750
  * 
- 1841 Router


# Ignore

AI Mode
All
Videos
Images
Shopping
More




# Resetting Passwords ## (badpassword) - Catalyst 3750 * - 1841 Router
Recovering password in Cisco Routers - GeeksforGeeksTo recover or reset forgotten passwords on a Cisco Catalyst 3750 switch or a Cisco 1841 router, you must connect via a console cable and interrupt the normal boot sequence. 

Cisco Community
 +1
Catalyst 3750 Switch Password Reset
To bypass a forgotten password on a Catalyst 3750 without permanently losing your settings, use the physical mode button. 

Layer23-Switch
 +1
Connect Console: Attach your console cable to the switch and open a terminal program (like PuTTY) set to 9600 baud.
Power Off: Unplug the power cord from the switch.
Hold Mode Button: Press and hold the Mode button on the front left panel while plugging the power cord back in.
Release Button: Keep holding the button until the System (SYST) LED turns solid green (or blinks amber briefly), then release it.
Initialize Flash: At the switch: prompt, type:
text
flash_init
Use code with caution.
Rename Configuration File: View your flash files and rename the configuration file to bypass the startup config:
text
dir flash:
rename flash:config.text flash:config.old
Use code with caution.
Boot the Switch: Type boot and press Enter. Skip the initial setup dialog by answering no.
Restore Configuration: Enter privileged EXEC mode, rename the file back, and pull the config into memory:
text
enable
rename flash:config.old flash:config.text
copy flash:config.text system:running-config
Use code with caution.
Change Password: Enter global configuration mode and set your new password:
text
configure terminal
enable secret [new-password]
end
Use code with caution.
Save and Reload: Save your running configuration to startup and reload:
text
copy running-config startup-config
reload
Use code with caution.
 

YouTube
·Network Wizkid
 +5
Cisco 1841 Router Password Reset
To recover a password on a Cisco 1841 router, you change the configuration register to ignore the startup configuration during boot. 

Cisco Community
 +1
Connect Console: Connect your console cable, set terminal emulation to 9600 baud, 8 data bits, no parity, 1 stop bit.
Power Cycle: Turn the router off using the power switch and turn it back on.
Interrupt Boot: Press the Ctrl + Break keys on your keyboard within 60 seconds of startup to enter the rommon 1> prompt.
Change Confreg: Type the following command to ignore startup configuration:
text
confreg 0x2142
Use code with caution.
Reset Router: Type reset to reboot the device.
Bypass Setup: Answer no to any initial setup prompts.
Load Startup Config: At the prompt, enter privileged mode and load the saved configuration into active memory:
text
enable
copy startup-config running-config
Use code with caution.
Update Password: Overwrite or add your new enable secret:
text
configure terminal
enable secret [new-password]
Use code with caution.
Restore Configuration Register: Set the boot register back to normal (0x2102):
text
config-register 0x2102
end
Use code with caution.
Save: Save your updated settings:
text
copy running-config startup-config
Use code with caution.
 

artofrf.com
 +6
Watch this step-by-step video guide to see a visual demonstration of the Cisco router password recovery process:
How to Recover a Password on a Cisco Router by Cisco Config on YouTube
5:50
How to Recover a Password on a Cisco Router
80K views · 4 years ago

YouTube · Cisco Config
Which device are you working on right now, the 3750 switch or the 1841 router? If you run into a specific prompt error, let me know and I can help you troubleshoot it.





Cisco Community
how to reset cisco router 1841 password
To reset the password on a Cisco 1841 router, you can try the following steps: 1. Connect the console cable to the router 2. Turn ...


YouTube
·
Network Wizkid
Cisco 3750-x Switch Password Recovery Procedure
hello everybody Kelvin here and my video today is focusing on a password recovery for a 3750X switch that is a Cisco switch. and t...

5m

Cisco Systems
Catalyst 3750-X and 3560-X Switch Software Configuration ...
Step 2 Set the line speed on the emulation software to 9600 baud. Step 3 On a Catalyst 3750-X switch, power off the standalone swi...
Show all
For the switch we're on Putty now
Great! Since you are on PuTTY, let's walk through the Catalyst 3750 password recovery process step by step.
Have you already unplugged the power cord from the back of the switch?
If not, go ahead and do that now. When you are ready, follow these next steps closely:
Press and hold the physical Mode button on the front left panel of the switch.
While still holding the button, plug the power cord back into the switch.
Keep holding that Mode button until the System (SYST) LED flashes amber and then turns solid green, then release it.
Once you do this, your PuTTY terminal screen should change and show a switch: prompt.
Let me know once you see the switch: prompt on PuTTY, or if the switch boots up normally by mistake so we can retry!




what do I do on PuTTY
Now that you are at the switch: prompt in PuTTY, type these exact commands one by one, pressing Enter after each line:
Initialize the flash file system:
text
flash_init
Use code with caution.
(Wait a few seconds for the switch to finish loading the flash layout).
View your files to confirm the config name:
text
dir flash:
Use code with caution.
(You should see a file named config.text in the list).
Rename the configuration file so the switch ignores it on boot:
text
rename flash:config.text flash:config.old
Use code with caution.
Boot the switch:
text
boot
Use code with caution.
The switch will now start booting up. Because we renamed the file, it will think it has no configuration and ask: Would you like to enter the initial configuration dialog? [yes/no]:
Type no and press Enter to get to a clean setup prompt.
Let me know once the boot finishes and you see the blank Switch> prompt!




No I just opened PuTTY what do i do
Ah, got it! Let's get PuTTY configured first so you can talk to the switch.
Before clicking anything else in PuTTY, make sure your console cable is plugged into the switch's console port and your computer's USB port.
Step 1: Configure PuTTY
Under Connection type, click the Serial radio button (on the far right).
In the Serial line box, type your COM port (e.g., COM3 or COM4). If you don't know it, let me know.
In the Speed box, type 9600.
Click Open at the bottom. A blank black window will pop up.
Step 2: Trigger Password Recovery Mode
Now, let's boot the switch into its special recovery mode:
Unplug the power cord from the back of the Cisco 3750 switch.
Press and hold the physical Mode button on the front left panel of the switch.
While keeping the button held down, plug the power cord back in.
Keep holding that Mode button until the SYST (System) LED on the front flashes amber and then turns solid green, then let go.
Look at your PuTTY window. It should now display text ending in a switch: prompt.
Do you see the switch: prompt on your screen, or did it ask you for a password again?




How do I find out the COM
To find your COM port number, you need to open Windows Device Manager.
Follow these quick steps:
Press the Windows Key + X on your keyboard and select Device Manager from the menu that appears.
Scroll down the list and click the arrow next to Ports (COM & LPT) to expand it.
Look for your cable, which usually says something like USB Serial Port, Prolific USB-to-Serial, or Silicon Labs.
Note the number in parentheses next to it—for example, (COM3) or (COM4).
What COM number do you see listed there? (If you don't see "Ports (COM & LPT)" at all, let me know!)




We have it open
Perfect! Now that PuTTY is open and connected to the switch, we can start the password bypass.
Take a look at your PuTTY window. Since you just opened it, the screen might be completely blank, or it might be full of text.
Let's get to the recovery mode:
Unplug the power cord from the back of the switch.
Press and hold the physical Mode button on the front left panel of the switch.
While holding that button down, plug the power cord back in.
Keep holding the Mode button until the SYST (System) LED on the front flashes amber and then turns solid green, then let go.
Once you release the button, look at your PuTTY screen. You should see some boot text ending in a prompt that looks exactly like this:
text
switch:
Use code with caution.
Did you successfully get to the switch: prompt?




Continue
Now that you are at the switch: prompt, type these exact commands one by one, pressing Enter after each line:
Initialize the flash memory:
text
flash_init
Use code with caution.
(Wait a few seconds for the switch to finish initializing).
Rename the configuration file so the switch ignores it on boot:
text
rename flash:config.text flash:config.old
Use code with caution.
Boot the switch:
text
boot
Use code with caution.
The switch will now start loading its boot software. Because we renamed the configuration file, it will think it has no settings and ask: Would you like to enter the initial configuration dialog? [yes/no]:
Type no and press Enter.
Let me know when the boot finishes and you see the blank Switch> prompt!




Okay we're good
Awesome. Now that you are at the blank Switch> prompt, we need to restore your old configuration so you don't lose any settings, and then set the new password.
Type these exact commands one by one, pressing Enter after each line:
Enter privileged mode:
text
enable
Use code with caution.
(Your prompt will change to Switch#)
Rename your old configuration file back to its original name:
text
rename flash:config.old flash:config.text
Use code with caution.
(Press Enter again to confirm if it asks you to)
Load the old configuration into the active memory:
text
copy flash:config.text system:running-config
Use code with caution.
(Press Enter to confirm the destination filename. Your original host name should now appear in the prompt, like YourSwitchName#)
Enter global configuration mode:
text
configure terminal
Use code with caution.
Set your new password (replace [your-new-password] with what you want to use):
text
enable secret [your-new-password]
Use code with caution.
Exit configuration mode:
text
end
Use code with caution.
Save everything so it sticks:
text
copy running-config startup-config
Use code with caution.
(Press Enter to confirm)
You are all done! Your password is reset and your configuration is saved.
Did the final copy running-config startup-config command complete successfully?




Can we test it?
Yes, absolutely! Let's test it right now to make sure you can log in with the new password.
Type these two commands in PuTTY to lock yourself out and log back in:
Exit privileged mode:
text
disable
Use code with caution.
(Your prompt will drop back down to YourSwitchName>)
Try to log back in:
text
enable
Use code with caution.
The switch will now ask you for a password. Type your new password and press Enter.
(Note: PuTTY will not show any characters or asterisks while you type the password—this is normal Cisco security behavior. Just type it blindly and hit Enter).
Did it let you back in and show the YourSwitchName# prompt?




We're all set!
That is awesome news! You have successfully recovered the switch without losing any of your configuration. Great job walking through those steps.
Are you ready to move on to the Cisco 1841 router, or is there anything else you need to verify on this switch first?




So can we safley get rid of the switch
Yes, you can safely disconnect and remove the switch now!
Since we ran the copy running-config startup-config command, your new password is permanently saved. You can unplug the power cable and the console cable from the switch whenever you are ready.
Are you moving on to resetting the 1841 router next, or are you all done with your network devices for today?




We're doing router now
Let's get the Cisco 1841 router sorted out.
Move your console cable from the switch over to the Console port on the back of the router. You can keep the same PuTTY window open.
Once you are plugged into the router, follow these steps to bypass the password:
Step 1: Trigger Rommon Mode
Turn off the router using the power switch on the back.
Turn it back on, and immediately start tapping the Ctrl + Break keys on your keyboard repeatedly.
If done correctly, the boot will stop, and your PuTTY screen will display a special prompt that looks like this:
text
rommon 1 >
Use code with caution.
(Note: If you are using a laptop or a keyboard without a Break key, you might need to press Fn + Ctrl + Pause/Break or Ctrl + Shift + Esc).
Step 2: Change the Boot Register
At the rommon 1 > prompt, type these two commands to force the router to ignore its password config on the next boot:
Change the register:
text
confreg 0x2142
Use code with caution.
Reboot the router:
text
reset
Use code with caution.
The router will now reboot. Because of the code we entered, it will bypass its old configuration and ask: Would you like to enter the initial configuration dialog? [yes/no]:
Type no and press Enter.
Let me know once you see the blank Router> prompt on your screen!




Wait is there a shutdown for the switch?
No, Cisco switches do not have a shutdown command or a power switch.
You can safely pull the power cord straight out of the back of the switch right now.
Because we already saved your work using the copy running-config startup-config command, your changes are completely safe and won't be lost when the power cuts.
Once you've unplugged it, go ahead and connect your console cable to the 1841 router and let me know when you are ready to start the Ctrl + Break sequence!




Okay now the router
Let's get this router password reset. Keep your PuTTY window open and make sure your console cable is plugged into the Console port on the back of the Cisco 1841 router.
Follow these steps to interrupt the boot process and bypass the password:
Step 1: Interrupt the Boot
Turn off the router using the physical power switch on the back.
Turn it back on, and immediately start pressing the Ctrl + Break keys on your keyboard repeatedly (within the first 60 seconds of booting).
Laptop tip: If your keyboard doesn't have a dedicated Break key, try holding Fn + Ctrl + Pause/Break or Ctrl + Shift + Esc.
If successful, the standard boot text will stop, and you will see this prompt:
text
rommon 1 >
Use code with caution.
Step 2: Change the Register and Reboot
Once you see that rommon 1 > prompt, type these two commands, pressing Enter after each:
Tell the router to ignore the configuration file on boot:
text
confreg 0x2142
Use code with caution.
Reboot the device:
text
reset
Use code with caution.
The router will now reboot. Because it is ignoring its old settings, it will act like a brand-new router and ask: Would you like to enter the initial configuration dialog? [yes/no]:
Type no and press Enter to pass it.
Let me know when the screen settles and you see the blank Router> prompt!




we closed putty on accident
No problem at all, that is easy to fix!
Just open PuTTY back up on your computer and configure it exactly like we did for the switch:
Click the Serial radio button on the far right.
In the Serial line box, type your COM port (e.g., COM3 or COM4).
In the Speed box, type 9600.
Click Open at the bottom.
Once your black PuTTY screen pops up, press the Enter key a few times.
What do you see on the screen now? Did the router finish booting to a Router> prompt, or do you see something else?




Okay we're back to the black screen
Perfect. Press the Enter key on your keyboard a few times to get a prompt to show up.
Depending on where the router is in its boot process, you will likely see one of two things:
Scenario A: It asks Would you like to enter the initial configuration dialog? [yes/no]:
If you see this, type no and press Enter.
Scenario B: It shows a blank Router> prompt.
Which one of those do you see on your screen right now? If you see a different prompt entirely (like rommon or a password request), let me know!




we just reset and are waiting for boot
Excellent, you timed that perfectly!
While it finishes loading, it will scroll a bunch of text on your screen. At the very end of the boot process, it will stop and ask you a question.
Look out for this exact line at the bottom:
Would you like to enter the initial configuration dialog? [yes/no]:
As soon as you see that pop up:
Type no and press Enter.
Press Enter one more time when it says "Press RETURN to get started."
Let me know when you successfully see the blank Router> prompt!




Okay we're there
Awesome! Now that you are at the blank Router> prompt, we need to load your old configuration back into memory so you don't lose any settings, change the password, and then fix the boot register so it boots normally next time.
Type these exact commands one by one, pressing Enter after each line:
Enter privileged mode:
text
enable
Use code with caution.
(Your prompt will change to Router#)
Load your old configuration into active memory:
text
copy startup-config running-config
Use code with caution.
(Press Enter to confirm the destination filename. Your original router name should now show up in the prompt, like YourRouterName#)
Enter configuration mode:
text
configure terminal
Use code with caution.
Set your new password (replace [your-new-password] with your chosen password):
text
enable secret [your-new-password]
Use code with caution.
CRITICAL STEP - Reset the boot register to normal mode:
text
config-register 0x2102
Use code with caution.
(If you skip this, the router will keep ignoring its config every time it power cycles).
Exit configuration mode:
text
end
Use code with caution.
Save your new password and settings permanently:
text
copy running-config startup-config
Use code with caution.
(Press Enter to confirm).
Let me know once you finish the final save. Would you like to test the new password to ensure you can log back in successfully?




the startup-config running-config gives us a "Non-violate configuration memory invalid or not present"
That error means the router does not have a saved configuration file in its memory. 
This happens if the router was previously factory reset, erased, or if it is a completely brand-new device out of the box. Because there is no old file to load, you cannot run copy startup-config running-config. 

Cisco Community
 +2
Since there are no old configurations or old passwords to preserve, you can skip that step entirely and configure it as a fresh router.
Type these exact commands to set your new password and fix the boot register:
Enter global configuration mode:
text
configure terminal
Use code with caution.
Set your new password:
text
enable secret [your-new-password]
Use code with caution.
Fix the boot register so it saves properly from now on:
text
config-register 0x2102
Use code with caution.
Exit configuration mode:
text
end
Use code with caution.
Save the configuration so it creates a brand new startup file:
text
copy running-config startup-config
Use code with caution.
(Press Enter to confirm).
Once you hit enter on that last command, it will build a brand new startup configuration file. Let me know if it gives you a successful [OK] message! 

Cisco Community





Cisco Community
3560 Can't run "copy startup-config running-config" ?? !! ??
Oct 11, 2005 — You might see this error when a router has booted from scratch and has not saved the running-config to startup-config. You can try...


Cisco Community
No startup config - Cisco Community
Jan 3, 2009 — Router#copy startup-config running-config. %% Non-volatile configuration memory invalid or not present. Router#. Router#sh startup...


Cisco Community
The NVRAM Verification Failed or "startup-config file open ...
Jun 22, 2009 — Resolution. You can try this workaround: First take a backup of the running-config to a TFTP server. Use the erase startup-config:
Show all
okay all good
Awesome! That [OK] message means the router has successfully created a brand new configuration file, and your new password is permanently saved.
Just like the switch, Cisco routers do not have a shutdown command. You can safely flip the physical power switch on the back of the 1841 to the Off position and unplug your cables.
Both of your devices are completely reset and ready to go. Is there anything else on your network equipment or any other devices you need help config-managing today?




test?
Let's do a quick reboot test to verify that everything sticks and your new password works perfectly on a fresh startup!
Flip the router's physical power switch to Off, wait about 5 seconds, and then flip it back to On.
Watch your PuTTY screen as it boots up:
Check the boot behavior: It should not ask you Would you like to enter the initial configuration dialog? anymore. It should boot straight to a normal prompt.
Access the prompt: Press Enter a few times until you see the blank Router> prompt.
Test the password: Type enable and press Enter.
Log in: Type your new password (remember, PuTTY won't show the characters as you type) and hit Enter.
Did it successfully drop you into the privileged Router# prompt?




password for router
If the router is asking for a password, type the exact password you chose a moment ago during step 2 of the setup (where you typed enable secret [your-new-password]).
As a reminder:
Type it blindly: PuTTY will not show any characters, dots, or asterisks on the screen while you type the password. This is a standard Cisco security feature.
Just type the password out completely and press the Enter key.
Did it accept the password and change your prompt to Router#?




How do I test that so we know the password saved
To test it and make sure it saved permanently, type these two commands into PuTTY:
Exit privileged mode:
text
disable
Use code with caution.
(Your prompt will change back down to Router>)
Log back in:
text
enable
Use code with caution.
The router will now ask you for the password. Type the new password you chose and press Enter.
(Note: PuTTY will not show any characters or dots on the screen while you type—just type it blindly and hit Enter).
If it lets you in and shows the Router# prompt, your password is successfully saved to the configuration memory! Did it work?
