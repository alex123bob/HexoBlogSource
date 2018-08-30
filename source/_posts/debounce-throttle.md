---
title: Simple Implementation of Throttle and Debounce
date: 2018-08-24 11:03:22
tags:
---

> Throttle and Debounce

In front-end, throttle and debounce are quite important for performance enhancement. Especially useful on events like `window.onresize` which will be triggered in high frequency.

I have just put a very very simple implementation underneath.

````Javascript
function debounce(func, wait) {
    var timer = null;

    return function (){
        var self = this,
            args = arguments;
        clearTimeout(timer);
        timer = setTimeout(function (){
            func();
        }, wait);
    }
}


function throttle (action, delay) {
    var startTime = 0;

    return function (){
        var currentTime = new Date();

        if (currentTime - startTime > delay) {
            action.apply(this, arguments);
            startTime = currentTime;
        }
    }
}
````