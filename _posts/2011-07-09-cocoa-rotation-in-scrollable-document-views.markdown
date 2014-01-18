---
layout: post
title: 'Cocoa: Rotation in Scrollable Document Views'
tags:
- programming
- cocoa
- ios
date: 2011-07-09 11:17:27.000000000 -07:00
---
<p>I&rsquo;m continuing work on my <a href="http://blog.brokenrobotllc.com/archive/11/2010">mac
project</a>, which
involves a scalable document that can be embedded in an NSScrollView.
I recently wanted to add orientation to the set of valid
transformations.  While I&rsquo;m sure there are more generic solutions that
involve setting a composite affine transformation on the underlying
CALayer, I wanted something that I could implement in an hour or so.
I ran into some issues that I thought I&rsquo;d write about in hopes of
saving other people some time.</p>

<p>When coding against the SDK for OS X Lion, NSView has a
<code>setFrameCenterRotation:</code> selector, which is very convenient for
rotating about the center of the view.  If you need to run against
older versions of OS X, you can use a combination of <code>setFrameOrigin:</code>
and <code>setFrameRotation:</code> to accomplish the same thing.  These rotations
don&rsquo;t affect the internal coordinate system of the NSView, but change
its frame in the superview &ndash; also very convenient when it comes to
performing layout in the rotated view.</p>

<p>I experimented with these for my document view&rsquo;s transformations and
found some quirks.  Firstly, I tried to simply rotate the
NSScrollView&rsquo;s <code>documentView</code>.  From my previous post, this is the
<code>HostView</code> object.  That didn&rsquo;t go quite as well as I had hoped: the
rotation occurred around the incorrect locus, and the scroll view got
very confused about positioning.  When I performed the rotation, the
document would jump to an odd place, and then when I moved the scroll
bars, the document would jump back to the origin.</p>

<p>After that, I attempted to rotate the <code>_docView</code> inside the <code>HostView</code>
object.  Success!  At least, initial success: while the rotation
occurred and the document stayed stable inside the scroll view, the
dimensions of the <code>HostView</code> object stayed the same and the <code>_docView</code>
was positioned incorrectly.  I found that the
<code>NSViewFrameDidChangeNotification</code> was still being raised.  When I
analyzed the newly calculated frame for the <code>HostView</code>, I found the
width and height were the same as the original, when it should have
been changed.</p>

<p>Changing the code in <code>updateFrame:</code> to detect the <code>_docView</code> in
landscape orientation and transforming the frame appropriately makes
the scroll view recalculate its scroll extents correctly.  There&rsquo;s
still the problem of the incorrectly positioned document.  I&rsquo;m not
sure why Cocoa does this, but when calculating the frame for a
rotation, the view would consistently get shifted by a size
proportional to the size of the view.  I had to manually adjust the
frame after the rotation had adjusted it.</p>

<p>After all is said and done, the new code looks like this.</p>
<script src="https://gist.github.com/visigoth/715903.js"></script>
