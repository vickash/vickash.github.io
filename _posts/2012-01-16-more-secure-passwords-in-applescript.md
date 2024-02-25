---
layout: single
excerpt_separator: <!--more-->
title: More Secure Passwords In Applescript
---

Sometimes an Applecript needs authentication. For example, this happens if you use `do shell script` to run a UNIX command with administrator priveleges:

    do shell script "chown -R Vickash " & ¬
                    quoted form of "/Users/Vickash/Desktop/My Files" ¬
                    with administrator privileges


A dialog box appears prompting the user for their password before the shell script runs. This is fine if your Applescript is always manually run by the user. But for Folder Actions, or any type of script that runs on a trigger, this can be a problem. What if the user isn't at the machine when the script is triggered? And if the script runs frequently, do you really want to ask the user for a password every time?

If you've come across this problem before, you know that one solution is to put the login credentials inline. However, the security problem here is obvious; anyone looking at the source code has the user's login credentials:

    do shell script "chown -R Vickash " & ¬
                    quoted form of "/Users/Vickash/Desktop/My Files" ¬
                    user name "Vickash" ¬
                    password "this is a test" ¬
                    with administrator privileges

Ideally, the password would be stored elsewhere, encrypted, and only accessed by the script when necessary. Luckily, OS X provides an application just for this. It's called Keychain Access, and is located in the Utilities subfolder.
<!--more-->

## Keychains

Keychain Access lets you create collections of credentials called 'keychains', which are stored encrypted on disk. OS X makes one for you by default. It's stored at ~/Library/Keychains/login.keychain,  and is protected with your login password. It's used for things like saved passwords in Safari, SSH keys, and anything an app want's to keep encrypted on disk.

## Setup A Keychain For Applescript

To keep things clean, I recommend creating a new keychain specifically for storing generic password items that you want to work with in Applescript. Here's how:

Open Keychain Access.app and go to File > New Keychain... You can name the keychain anything and store it anywhere, but take note of its full path. You'll need that for the script to access it later.

![Create a new keychain.](/images/2012-01-16-more-secure-passwords/01.png)

2) You'll need to set a password for the keychain. This is separate from the credentials you want to store inside the keychain; it just restricts access to the keychain itself. Think of it like a LastPass or 1Password master password. 

![Set a password for the keychain.](/images/2012-01-16-more-secure-passwords/02.png)

3) Once the keychain is ready, create a generic password item by clicking on the "add" button below the empty list. Put in the credentials that you would have typed inline in the script before, and give the item a name. I've called it "Admin" here. It doesn't matter what you call it, but meaningful is better, and you'll need to know the name for the script to find the right item later.

![Create a keychain item for the credentials you want to secure.](/images/2012-01-16-more-secure-passwords/03.png)

4) To make things more convenient for the user, I recommend changing the keychain's settings so it doesn't automatically lock on sleep or after a period of inactivity. You can adjust this to your needs, but the idea is that the user will grant Applescript access to the keychain once, and then doesn't need to do so again until the next time he logs in.

![Edit the keychain settings.](/images/2012-01-16-more-secure-passwords/04.png)

![Set it so the keychain doesn't relock until the user logs out.](/images/2012-01-16-more-secure-passwords/05.png)

## Access Keychain Data From Applescript

We've solved half the problem. The password is stored in an encrypted form on disk, rather than in plaintext. But now we need some way to pull those credentials out of the keychain and use them in our script.

Apple used to provide a standard extension for this. You'd call `tell application "Keychain Scripting"` and be able to access the keychains, but I'm not sure that still exists in Lion, and it had major performance issues before that.

Daniel Jalkut at [Red Sweater Software](http://www.red-sweater.com/blog/) has written a tiny scriptable app called "Usable Keychain Scripting" for [Lion](http://www.red-sweater.com/blog/2035/usable-keychain-scripting-for-lion) and [Pre-Lion](http://www.red-sweater.com/blog/170/usable-keychain-scripting) systems. It's free, I've used it, and it works great. If you don't have a problem with installing something extra to get your scripts to run, head over to his blog and check it out.

If you'd like your scripts to not have external dependencies, like I've needed on a couple occasions, this won't work. I've come up with a pure AppleScript workaround. On OS X there's a command line tool `security` that can be used from the terminal to access keychains.

You can read more about `security` [here](http://developer.apple.com/library/mac/#documentation/Darwin/Reference/Manpages/man1/security.1.html), but we're mainly interested in is its `find-generic-password` option.

If I run:
    
    security 2>&1 find-generic-password -gs \
             "Admin" "/Users/Vickash/Library/Keychains/ScriptingDemo.keychain"
    
It returns:

    keychain: "/Users/Vickash/Library/Keychains/ScriptingDemo.keychain"
    class: "genp"
    attributes:
      0x00000007 <blob>="Admin"
      0x00000008 <blob>=<NULL>
      "acct"<blob>="Vickash"
      "cdat"<timedate>=0x32303132303131353136343231355A00 "20120115164215Z\000"
      "crtr"<uint32>=<NULL>
      "cusi"<sint32>=<NULL>
      "desc"<blob>=<NULL>
      "gena"<blob>=<NULL>
      "icmt"<blob>=<NULL>
      "invi"<sint32>=<NULL>
      "mdat"<timedate>=0x32303132303131353136343231355A00 "20120115164215Z\000"
      "nega"<sint32>=<NULL>
      "prot"<blob>=<NULL>
      "scrp"<sint32>=<NULL>
      "svce"<blob>="Admin"
      "type"<uint32>=<NULL>
    password: "this is a test"
    
It's obvious that the only lines we need are `"acct"<blob>="Vickash"` and `password: "this is a test"`. By executing the shell command via AppleScript, and writing a separate handler to extract the data we need, I've come up with the following:

    on extractData(theText, theFieldName, theEndDelimiter, spaces)
      set theDataStart to the offset of theFieldName in theText
      if theDataStart = 0 then
        return ""
      else
        set theDataStart to theDataStart + (length of theFieldName) + spaces
        set theData to text theDataStart through end of theText
        set theDataEnd to ((offset of theEndDelimiter in theData) - 1)
        set theData to text 1 through theDataEnd of theData
      end if
    end extractData

    on getCredentials of theKeychainItem from theKeychain
      set theKeychainPath to (POSIX path of theKeychain) as text
      try
        set theKeychainAlias to (POSIX file theKeychainPath) as alias
        set theKeychainPath to (POSIX path of theKeychainAlias)
      on error
        return "Keychain file not found at specified location: " & ¬
               theKeychain as text
      end try
      try
        set theResult to do shell script ¬
                         "security 2>&1 find-generic-password -gs " & ¬
                         theKeychainItem & " " & ¬
                         quoted form of theKeychainPath
        set theAccount to extractData(theResult, "\"acct\"<blob>=\"", "\"", 0)
        set thePassword to extractData(theResult, "password: \"", "\"", 0)
        return {account:theAccount, password:thePassword}
      on error
        return "Generic password item specifeid does not exist in keychain: " & ¬
               theKeychainItem
      end try
    end getCredentials
    
You can simply copy and paste these handlers into your script. Now, whenever your script script needs to get credentials from a keychain, do something like: 

    set theCredentials to 
        getCredentials of "Admin" ¬
        from "/Users/Vickash/Library/Keychains/ScriptingDemo.keychain"
    
This will return an AppleScript record. In my case:

    theCredentials is equal to {account: "Vickash", password: "this is a test"}
    
Now that the script has the login credentials stored in a record, I can execute the command that would have required manual authentication, like:

    do shell script "chown -R Vickash " & ¬
                    quoted form of "/Users/Vickash/Desktop/My Files" ¬
                    user name (account of theCredentials) ¬
                    password (password of theCredentials) ¬
                    with administrator privileges
    
Which is much more secure than:

    do shell script "chown -R Vickash " & ¬
                    quoted form of "/Users/Vickash/Desktop/My Files" ¬
                    user name "Vickash" ¬
                    password "this is a test" ¬
                    with administrator privileges
    
## Notes
 
* The first time the script runs, it will prompt you for the keychain password to unlock it. Put the password in and click "Always Allow" to give the script access to the keychain until it relocks itself, or you log out. Click "Allow" if you want to be prompted to unlock the keychain each time the script runs.

* If you've used the settings I recommended, the keychain won't relock unless it's manually relocked via the Keychain Access app, or until you log out.

* The `getCredentials` handler returns text, rather than a record on error, so you can write your script to catch this and display a dialog box.