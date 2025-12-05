---
layout: post
published: false
date: 2025-12-05
title: Creating Hostly with Kotlin Multiplatform
---
At Hostly we're using Kotlin Multiplatform

Pain points:

iOS

*   Very slow sync time with multi-module
    
*   Relying on SPM or Cocoapods means trying to figure out a very specific setup that seems to have issues all of the time
    
    *   Needing to rely on SPM4KMP a 3rd party library to meaningfully be able to develop
        
*   Debugging is very difficult to do
    
    *   Should be able to be mitigated with `experimental Multiplatform IDE feature` in the IDE settings (not fully)
        
*   Updating Xcode or your OS may sometimes break compatibility
    

Kotlin

*   The community feels small
    
    *   Most discussion happens in the Kotlin lang slack
        
*   Not many 3rd party libraries, and even the ones that do exist go abandoned which leaves the pool even smaller
    
*   When you get into an issue for your specific usecase, you may be the first person to experience that issue
    
*   No test fixtures cause friction
    
*   On top of test fixtures, having to use different methods per source set makes it difficult to keep track of what you need to do per source set. androidMain, androidDebug, but no iosDebug?
    
    *   The solution has been to use build config a 3rd party library
        
*   Deeplinks and navigation tends to be a pain point where you have to roll your own solution or use a 3rd party