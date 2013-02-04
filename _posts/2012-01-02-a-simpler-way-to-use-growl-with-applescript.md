---
layout: default
title: A Simpler Way To Use Growl With Applescript
---

#A Simpler Way To Use Growl With Applescript
2012-01-02

[Growl](http://www.growl.info) is a great pop-up notifier that many Mac applications integrate with. If you write Applescripts for yourself or others, it's useful for displaying pop-up notifications, rather than dialog boxes, notifying users when a script runs on a schedule, or a trigger, or even to send users reminders.

However, Growl suffers from a wordy AppleScript implementation. Each time you want to send a new notification, you have to register your application with Growl, register the notification with Growl, and then display it. Also, there are a bunch of version-incompatible ways to access the scriptable Growl object.

This is an attempt to wrap all of this wordiness up into a single AppleScript handler which will let you create Growl notifications with a single line.

##How To Set It Up

Step 1: Make sure you have Growl version >= 1.2.2 on your Mac. This might work with older versions, but I haven't tested it.

Step 2: Copy the following property and handler into the Applescript that you want to send Growl notifications from:

    property allNotifications : {}
    on growl(theTitle, theContent)
      if allNotifications does not contain theTitle then
        set allNotifications to allNotifications & theTitle
      end if
      set myName to path to me as text
      set TID to text item delimiters
      set text item delimiters to ":"
      set myName to the last text item of myName
      set text item delimiters to TID
      tell application id "com.Growl.GrowlHelperApp"
        register as application myName ¬
                    all notifications allNotifications ¬
                    default notifications allNotifications ¬
                    icon of application "AppleScript Editor"
        notify with name theTitle ¬
                    title theTitle ¬
                    description theContent ¬
                    application name myName
      end tell
    end growl


Step 3: When you want your script to send a notification, call the `growl` handler, passing in the notification title and content as arguments:

    growl("Notification Title", "This is the content of the notification.")

That's it.

## How It Works

The `allNotifications` property is a list of all the notifications your script has already registered with Growl. Each time you call the `growl` handler, it checks to see if the title you passed in is in the list. If it isn't, it gets added to the list, and registered with Growl.

Next, we get the file path to your script and extract the filename. This is used as the 'application name' to register the script with Growl. The `register` line does the registering, telling it to use AppleScript Editor's icon for the notification. The `notify` line displays the actual notification.

## Notes

* `allNotifications` is an AppleScript property, which persists across runs, but resets to an empty list when you edit or recompile the script. This means that, if you've set up custom styles or priorities in Growl for your script's notifications, they get reset if you edit or recompile.   
In most cases, this won't matter, but if it does to you, you can modify the handler to store the list of notifications in a text file in the same folder as the script, and read it in on each run. Overkill for my needs, but definitely a workable solution.

* You can replace 'AppleScript Editor' with the name of any app whose icon you'd like to use for the notification.

* You'll notice that I've addressed the Growl application by its id and not by its name. This is because `tell application "Growl"` works with version 1.3 and up only, but `tell application "GrowlHelperApp"` works with version 1.2.2 and lower only.  
I've found that by avoiding names altogether, and addressing it via id, the script works with all versions. Scripts you create with this handler should work regardless of which version of Growl is installed on the user's system.

## More About Growl

Growl used to be an OS X preference pane, but with version 1.3, it moved to the Mac App Store, so it's now a $1.99, 64-bit, Lion-only application. Version 1.2.2 is still available for free at [www.growl.info](http://www.growl.info/downloads) It states compatibility with 10.5 and 10.6, but works fine on 10.7 as well. Version 1.3 adds some nice features, and I like that it's an application rather than a preference pane, but the core functionality of both is mostly identical for now.
