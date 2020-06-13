---
layout: post
title:  "TIL: Exponential Backoff Retries"
date:   2020-06-13 19:00:00 +0800
categories: [blog]
tags: [design pattern,http call,TIL]
---

There was an issue that has been bugging our team for the past few months. We were calling Google API on one of the services that was used internally. However it kept on failing.

After some time googling I found out that my team are not the only one facing the issue. Seems too me we need to learn to deal with possible failure that may occur.

- [Frequently http 500 internal error with google drive API drive.files.get](https://stackoverflow.com/questions/12471180/frequently-http-500-internal-error-with-google-drive-api-drive-files-get)

One of the replied in the stackoverflow [suggested the use of exponential backoff and retry](https://stackoverflow.com/a/12474984/5057488) pattern in handling the API call. Immediately I spend some time googling it however found out there's not much discussion of this design pattern.

The idea is simple - keep on retry the function with a exponential delay until it succeed or the maximum timeout reached.

[DZone.com](https://dzone.com/articles/understanding-retry-pattern-with-exponential-back) summarized the method as follows:

1. Identify if the fault is a transient fault.
2. Define the maximum retry count.
3. Retry the service call and increment the retry count.
4. If the calls succeeds, return the result to the caller.
5. If we are still getting the same fault, Increase the delay period for next retry.
6. Keep retrying and keep increasing the delay period until the maximum retry count is hit.
7. If the call is failing even after maximum retries, let the caller module know that the target service is unavailable.

I'm gonna try it out, wish me luck! **fingers crossed**
