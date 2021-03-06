---
layout: post
title: 'Immortal Objects - Or: How to Find Memory Leaks'
created: 1208796840
author: lior.kanfi
permalink: /devops/immortal-objects-or-how-find-memory-leaks
tags:
- DevOps
---
<p><span class="thmr_call" id="thmr_51"><span class="thmr_call" id="thmr_6"><p>This is a classic memory leak: Select <em>Window -&gt; New Window</em> in your Eclipse IDE and then close the new window right away. The heap usage grows a little bit. Open and close a couple new windows and the heap usage grows more. That&rsquo;s what bug <a href="https://bugs.eclipse.org/bugs/show_bug.cgi?id=206584">206584</a> is about. In this blog, I will use the &ldquo;New Window&rdquo; leak to explain how to find memory leaks using the <a href="http://www.eclipse.org/mat/">Memory Analyzer</a>.</p> <p>A <a href="http://en.wikipedia.org/wiki/Memory_Leak">memory leak</a> is an unintentional memory usage. In Java programs, leaks are objects which are not used/needed anymore, but which are still reachable and therefore are not removed by the <a href="http://en.wikipedia.org/wiki/Garbage_Collection">Garbage Collector</a>. In our case this means that instances of <em>WorkbenchWindow</em> are not garbage collected even though the window is closed and the workbench window instance is not needed anymore.</p></span></span></p>
