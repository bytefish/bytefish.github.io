title: File Lock Check 
date: 2026-04-17 20:50
tags: wpf, dotnet
category: dotnet
slug: file_lock_check
author: Philipp Wagner
summary: This article introduces FileLockCheck. 

**FileLockCheck** is an application to show, which processes are locking a folder or file. This 
is useful to find out why we cannot delete a folder or a file... double so, when trying to delete 
a monstrous `node_modules` folder. 🤭

I have released the MSI installer to GitHub at:

* [https://github.com/bytefish/FileLockCheck/releases/download/1.0.0/FileLockCheck.Setup.msi](https://github.com/bytefish/FileLockCheck/releases/download/1.0.0/FileLockCheck.Setup.msi)

## Basic Usage ##

Start the **File Lock Check** application using the Desktop Shortcut or using the executable.

You'll see a new tray icon appears:

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/file_lock_check/tray_file_lock_check.jpg">
        <img src="/static/images/blog/file_lock_check/tray_file_lock_check.jpg">
    </a>
</div>

Now just select a folder or file in Explorer and press `Control + Shift + U`. 

You will then see all processes locking the folder or file:

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/file_lock_check/window_file_lock_check.jpg">
        <img src="/static/images/blog/file_lock_check/window_file_lock_check.jpg">
    </a>
</div>

You can switch the language or quit the application using a Right Click on the Tray Icon:

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/file_lock_check/tray_right_click_file_lock_check.jpg">
        <img src="/static/images/blog/file_lock_check/tray_right_click_file_lock_check.jpg">
    </a>
</div>

To change the Global Shortcut click into the Shortcut Box and press your desired Key combination:

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/file_lock_check/window_change_shortcut.jpg">
        <img src="/static/images/blog/file_lock_check/window_change_shortcut.jpg">
    </a>
</div>

## Feedback ##

I would appreciate a little feedback and feel free to raise issues or ask for features.