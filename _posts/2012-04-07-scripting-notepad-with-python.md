---
layout: post
title: "Scripting Notepad++ with Python"
date: 2012-04-07 22:14:00 +0530
tags: [python]
---
In this post I am going explain how Notepad++ can be scripted with Python. I came to know about this as I wanted to analyse huge amount of console logs.

## Installing Notepad++ Python Plugin
First we need to install Notepad++ Python plugin to able to control Notepad++ from python code. The plugin can be installed through Notepad++'s Plugin Manager by installing "Python Script" plugin or we can download the plugin from here http://sourceforge.net/projects/npppythonscript/ and extract the files into Notepad++'s plugin directory. As of this writing the Python Script plugin version is 0.9.2.

## Invoking Python Scripts
The Notepad++ Python scripts needs to placed in particular directory so that it will recognized by the Python plugin and can be invoked from Notepad++. Usually the directory is %APPDATA%\Notepad++\plugins\config\PythonScript. The scripts can be invoked through menu Plugins-&gt;Python Script-&gt;Scripts. We can also create toolbar button for these scripts to quickly invoke them.

## Counting Words Programmatically
To demonstrate the plugin lets write python script to count the number of characters, words and lines in the current editor window of Notepad++.
```python
from Npp import *
import re

numChars = 0
numWords = 0
numLines = 0

editorContent = editor.getText()
for line in editorContent.splitlines():
  numLines += 1
  for word in re.findall("[a-zA-Z0-9]+", line):
    numWords += 1
    numChars += len(word)

notepad.messageBox("Number of characters: %d \nNumber of words: %d \nNumber of lines: %d" % (numChars, numWords, numLines))
```
at line 8 we get the active editor window's text content and everything else is typical Python program except at line 15 we tell the number of chars, words and lines through Notepad++ message box.
## Bookmarking Programmatically
Lets see an another Python script which utilizes the bookmark feature of Notepad++
```python
from Npp import *

notepad.menuCommand(MENUCOMMAND.SEARCH_CLEAR_BOOKMARKS)
linesBookmarked = []

def onMatch(lineNumber, match):
  if lineNumber not in linesBookmarked:
    lineStartPos = editor.positionFromLine(lineNumber)
    editor.gotoPos(lineStartPos)
    notepad.menuCommand(MENUCOMMAND.SEARCH_TOGGLE_BOOKMARK)
    linesBookmarked.append(lineNumber)

editor.pysearch("Pos", onMatch)
```
the above script bookmarks all the lines that contains the word "Pos". The Editor class provides a method "pysearch" which can search for given regular expression and will call the given function for each match. Like the "pysearch" method the Editor and Notepad class object provides various helper methods to automate Notepad++ functionalities from Python script.

***Update on 02 July 2016***
I came to know that the above script is not working in the latest version of Python Script plugin(thanks to Robbin Harrell). Upon investigating I realized there were many backward incompatible changes introduced in the recent versions of Python Script plugin. The below is the updated version of the script that works with Python Script 1.0.6.0. As I no longer use Microsoft Windows or Notepad++ on daily basis, I might not able to realize any backward incompatible changes in Python Script and keep the script up to date. So please let me if the script doesn't work for you, I will try to fix it and keep it up to date.

```python
from Npp import *
 
notepad.menuCommand(MENUCOMMAND.SEARCH_CLEAR_BOOKMARKS)
linesBookmarked = []
 
def onMatch(match):
  lineNumber = editor.lineFromPosition(match.start())
  if lineNumber not in linesBookmarked:
    lineStartPos = editor.positionFromLine(lineNumber)
    editor.gotoPos(lineStartPos)
    notepad.menuCommand(MENUCOMMAND.SEARCH_TOGGLE_BOOKMARK)
    linesBookmarked.append(lineNumber)
 
editor.research("Pos", onMatch)
```