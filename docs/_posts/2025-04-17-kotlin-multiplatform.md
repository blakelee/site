---
published: true
date: 2025-04-17
title: Kotlin Multiplatform
---
The main purpose of this blog is to post about the random things in Kotlin Multiplatform that I found difficult. Since Kotlin Multiplatform alongside Compose Multiplatform are actively changing there are some hiccups that appear from time to time.

Here are some thing to note:

*   iOS dependencies with Cocoapods or Swift Package Manager (SPM) have been a huge headache
    
*   Finding the right dependency injection library (So far I've settled on Kotlin-Inject) since Dagger isn't KMP
    
*   Test fixtures aren't really a thing yet
    
*   Gradle build times
    
*   Gradle convention plugins
    
*   Running automation tests on Android and iOS
    

As things pop up I intend to write a small post about what the problem was and what the solution is. With how difficult it was to create a Kotlin Multiplatform convention plugin, I may write about that first since that seems to be the least documented part that is almost immediately a bottleneck once you create a project with 10+ modules and need to make it easy to create a new module.