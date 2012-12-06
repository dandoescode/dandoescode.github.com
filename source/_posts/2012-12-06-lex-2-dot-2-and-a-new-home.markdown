---
layout: post
title: "Lex 2.2 and a New Home"
date: 2012-12-06 14:21
comments: true
categories: [Lex, PHP]
---

Lex 2.2 has just been released.  Not much has changed, but a Minor version bump was required due to a backwards-incompatibility issue.

#### Changelog

* Fixed a test which was PHP 5.4 only.
* Added PHPUnit as a composer dev requirement.
* Added a Lex\ParsingException class which is thrown when a parsing exception occurs.

The 3rd point is the one that caused the version bump.  Before it would simply dump a message and then `exit()` (that was stupid).

### New Home

Lex has been moved from the Fuel GitHub organization to the PyroCMS organization ([https://github.com/pyrocms/lex]()).  This was done because it was originally built for PyroCMS and it will be maintained by myself and the Pyro team.

With this change I have also re-named the Composer package to `pyrocms/lex`, so please update your composer.json files. If you try using the old one, it should tell you that it is replaced by the new one and to update your requirement.

As always if you run into issues go ahead and let me know via the Issue Tracker.
