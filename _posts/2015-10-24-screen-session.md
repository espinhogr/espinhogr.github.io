---
title: "Never run a script in background again: screen"
category: neverforget
tags:
 - shell
 - linux
 - utilities
 - command
 - ssh
---

## The problem

You need to run your `time-consuming.sh` script. Being a smart human being you decide to `ssh` in your dev VM and run the task there because either you don't want to use your precious local resources or you have a limited bandwidth (you prefer to waste them listening to the music on YouTube in 4k quality):

    $ time ./time-consuming.sh

and it starts. Of course you want to know how long it takes to run because you have to do the same on the production environment, that's why you use `time` (You don't want me to tell your mother you run the scripts straight in the production environment, do you?). You check the log on the `stdout` and everything looks fine. "Let me check the resources consumption with `top`, I want to see how it performs". After this thought you face the first problem: your `ssh` session is busy with your script running. Easy problem to solve, you split your _console_ view and you open another `ssh` so that you can monitor your amazing script.

It's 2.30pm and it's done, it took only 2h hence you can run it in production and you are perfectly within the deadline set at 5.30pm today. You run it and you go back to the ticket you were working on. At one point the music in your earphones stops, the WiFi dropped out:

    Write failed: Broken pipe
    $ _

Ohhhh come on, really? 1h58m wasted (everybody knows that the connection always drops out 2m before the end of the process). You missed the deadline.

> _The common user knows that he can use_ `&` _to run the script in background. This solves the problem of opening another ssh session to check the performance and also the "broken pipe" one, but you lose track of the log. The expert user knows that using_ `nohup` _you redirect the output by default in a file_ `nohup.out`_._ _This is a good solution but doesn't solve the problem of switching between_ `tail -f nohup.out` _to check the logs and all the other commands you are running on your dev VM._

This is only one example of the problems you can face while using `ssh`.

## The solution:

TL;DR `screen` command

`screen` is a really powerful command which creates a sort of console manager. It allows you to have more than one console session within the current console and all of them are alive until you kill them.

Just remember, when you use `screen` you have only one true god: `ctrl + a` (aka `C-a`). This is what has to precede every single command you want to give to your console manager.

Now let's see how to solve the problem that stopped us from meeting the deadline.

### Creating a screen session

First of all, after you opened your `ssh` session, you want to open a new `screen` session on your remote VM:

    $ screen

this shows you the license, you just have to press `space` or `enter`. At this point it looks like nothing happened, you still have your cursor as if you didn't issue any command. Actually you are already inside a session, you can confirm it pressing `C-a` two times, you'll see the warning `No other window.` in the bottom-left part of the screen.

We are in, we can finally run our script:

    $ time ./time-consuming.sh

### Opening a new window

Now we have a script running in the current window and we can see the log printed out on the `stdout`. It's time to see how our script performs. Press `C-a c` to create a new window and you'll see a new empty console opened. In this moment the script is still running in the background in the other window.

    $ iostat

and we check how the `I/O` is reacting to our script.

### Switching among windows

Now that you've checked the performance, you want to go back and see which point your script is at. There are several ways of doing it but the easiest one to start is `C-a "`, this gives you a menu where you can navigate with the arrows and select with `return`. If you want to know more ways of moving among the windows `man screen` is your friend.

### Detaching from a session

You can detach in two ways from a session:

- _gracefully:_ you press `C-a d` and you are back in the real console. No worries, you didn't kill your `screen` session therefore your script is still running.
- _the hard way:_ you pull the ethernet cable from your computer (or you close your laptop). This way also the `ssh` session is closed (depending on your configuration) but your script is still fine.

> _Usually "the hard way" happens without the explicit user approval (i.e. connection drops out) even though I know many braves._

### Attaching to the last existing session

The `broken pipe` error has happened again, but this time you were smarter than it, you created a `screen` session. Now you can easily `ssh` into your VM and issue

    $ screen -rd

and you have back the same windows you had before, one with your script running and the other to monitor the performance.
Before running this command sometimes you want to check if there is any open session:

    $ screen -ls
    There is a screen on:
	    42392.ttys001.YourVM	(Detached)
    1 Socket in /var/folders/9d/3zcqlq6s79s3jrp3jsx3h0y90000gn/T/.screen.
    $ _

and as you can see there is one.

### Kill a window

When you're done, to kill a window you simply switch on that window and you press `C-a k`. When all the windows are killed the session is closed.

## Conclusion

As you can see, `screen` command solves many issues when working on a remote machine. I'm pretty sure from now on you're not going to use `&` and `nohup` again in this case.  
