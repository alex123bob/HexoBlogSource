---
title: regularexpression-reverse-pattern-search
date: 2018-08-30 09:35:19
tags:
---

### A RegExp reverse pattern search in Javascript:

My colleague asked me for a solution on manipulating String, simply replacing all backslash with empty string, but only match backslash followed by anything but a backslash '\'. This is definitely a reverse pattern search case.

> Solution

```` Javascript

    function removeStandaloneBackslash (str){
        var reg = /\\(?=[^\\])/g;
        return str.replace(reg, '');
    }

    removeStandaloneBackslash("\\test\\\\abcdefg\\!@#$%^&*()");

````

