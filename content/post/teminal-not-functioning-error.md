+++
categories = ["how-to"]
comments = false
date = "2015-10-13T23:00:20+01:00"
draft = false
showcomments = true
showpagemeta = true
slug = ""
tags = ["term", "cli"]
title = "Teminal Not Functioning Error"

+++

### Symptoms
try and run a command and get this:
```
WARNING: terminal is not fully functional
```

### Cure
It is to do with your TERM enviroment setting.
Running `echo $TERM` will return something like `screen-256color` this needs to be changed. I have had success with the following:
```
export TERM=ansi
```
You should also Change this in your .bashrc/.zshrc if after a reboot you get the same issue.

### References

- [http://superuser.com/questions/575844/warning-terminal-is-not-fully-functional](http://superuser.com/questions/575844/warning-terminal-is-not-fully-functional)

