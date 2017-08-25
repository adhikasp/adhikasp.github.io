---
title: "GSOC Phase 3 - Polishing aspect feature"
layout: single
excerpt: "In this phase, I try to polish up feature so hopefully coala's
aspect will became more sane to be used."
sitemap: true
author_profile: false
category: "developer"
tags:
  - "English"
  - "coala"
---

This post is part of my GSOC journey in coala.

Because aspect is still new and experimental, information and other support
about it is stil. Also, not all bear in coala (read: none of them) support
aspect. And it will be a long time before it happen. So I think it's important
to make the migration process became more easy.

In this phase, I try to polish up feature so hopefully it will became more
sane to be used. Of course there are still many missing things that I don't
think of now, but hopefully I can catch them and add follow up patches. The
polishing up could be separated to users perspective and developer perspective:

## User's perspective

### Warn unfulfilled aspect

As mentioned before, aspect ready bear is still a rare species compared to
number of aspect definition in coala. This make it very possible to request an
aspect that doesn't have any bear that support it. We need to warn user if this
happen.

<https://github.com/coala/coala/pull/4657>

Funny fact I just learn that [`for...else` syntax](http://book.pythontips.com/en/latest/for_-_else.html)
is a thing!

### Logs Various Message

Make sure users is informed what happen (if something goes wrong). Cases like
setting `bears` and `aspects` setting at the same time will ignore the `bears`
setting and missing or invalid `language` setting should throw a warning logs.

<https://github.com/coala/coala/pull/4582/files>

I also use this opurtunity to refactor the verification code into its own
function instead of separated into various function in order to make it more
easier to read (hopefully).

### Write Docs on Configuring coala With Aspect

I write a help page on how to configure aspect and list of required setting.

<https://github.com/coala/documentation/pull/462>

### Create Example Project that Showcase it All

Showcase aspect in progress, best tutorial is a live project!

WIP, should be possible after all PR is merged

## Developer perspective

### Write Docs on Bear's Aspectization

A docs containing example of a aspectized bear

<https://github.com/coala/coala/pull/4666>

### Create Mapping from Bear's Setting to Aspect/Taste Equivalent

This is a bit of quality-of-life feature but still pretty important to make
sure bear got aspectized easily. This will map a bear setting with aspect.

<https://github.com/coala/coala/pull/4662>

### Integration Test

In here I write a test that run `coala` with aspect configuration to make sure
everything work correctly.
