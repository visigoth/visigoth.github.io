---
layout: post
title: Using NSPopover with NSStatusItem
tags:
- osx
- cocoa
- programming
date: 2012-04-21 18:22:00.000000000 -07:00
---
<p>Mac OS X Lion supports a new Cocoa object called <code>NSPopover</code>.  This object
acts a lot like the popover introduced in CocoaTouch on iPad.  There are
lots of places normal applications can use <code>NSPopover</code> naturally.  I
recently started working on a personal project that uses an <code>NSStatusItem</code>,
and wanted to use an <code>NSPopover</code> to manage the primary user interface for
the content shown when interacting with the <code>NSStatusItem</code>.</p>

<p>Unfortunately, the interactions between a hidden object called
<code>NSPopoverWindow</code> and Mac OS X&rsquo;s <code>NSStatusBarWindow</code> create some problems.
 On top of this, having an <code>NSPopover</code> where the content view contains edit
fields creates further problems.</p>

<p>However, I was finally able to get the interactions working correctly (at
the expense of using the built-in <code>NSPopoverBehavior</code> styles).  This post
takes you through the problems and describes my solutions.</p>

<p>Yes, I could have used the <a href="https://github.com/shpakovski/Popup/tree/master/Popup">Popup</a> project, but I just
wanted to make <code>NSPopover</code> work.</p>

<h2>Initial Code</h2>

<p>The initial code is pretty simple: just create a custom view and attach it
to an <code>NSStatusItem</code> object.  Even though I did not need more than the
functionality provided by the default <code>NSStatusItem</code> object, <code>NSPopover</code>
requires a view for the <code>showRelativeToRect:ofView:preferredEdge:</code>
selector.  I couldn&rsquo;t figure out a way to access the <code>_fView</code> member of the
<code>NSStatusItem</code> that&rsquo;s visible in the debugger, so I was stuck
re-implementing the basic <code>NSStatusItem</code> view functionality of an image
that flips when activated.  I call this <code>BRStatusItemIconView</code>.  The code
in the application delegate looks like this:</p>

<div class="CodeRay">
  <div class="code"><pre>NSStatusItem* statusItem = [[NSStatusBar systemStatusBar]
  statusItemWithLength:32];
[statusItem setHighlightMode:YES];

_iconView = [[BRStatusItemIconView alloc]
  initWithStatusItem:statusItem];
_iconView.image = [NSImage imageNamed:@&quot;Status&quot;];
_iconView.highlightedImage = [NSImage imageNamed:@&quot;StatusHighlighted&quot;];</pre></div>
</div>


<p>The <code>initWithStatusItem:</code> selector creates the necessary subviews (since
it&rsquo;s not loaded from a nib) and attaches itself to <code>statusItem</code> through
<code>setView:</code>.</p>

<p>I used a separate object to be the popover controller and attach itself to
the <code>NSStatusItem</code>.  Since I used a custom view, the <code>action</code> and <code>target</code>
properties of the <code>NSStatusItem</code> couldn&rsquo;t be used.  I created a
<code>BRStatusItemIconViewDelegate</code> protocol that had one selector:</p>

<div class="CodeRay">
  <div class="code"><pre>- (void)activated:(BRStatusItemIconView*)sender;</pre></div>
</div>


<p>When the icon was clicked, it would invoke this selector on a delegate.  My
<code>BRStatusItemPopoverController</code> object is the popover&rsquo;s controller and
attaches itself to the <code>BRStatusIconView</code> created in the application
delegate.  The popover was initialized like this:</p>

<div class="CodeRay">
  <div class="code"><pre>_popover = [[NSPopover alloc] init];
_popover.behavior = NSPopoverBehaviorApplicationDefined;
_popover.contentViewController = viewController;
_popover.delegate = self;
_popover.animates = NO;</pre></div>
</div>


<p>Note I used <code>NSPopoverBehaviorApplicationDefined</code>, the importance of which
will be discussed later.  The <code>viewController</code> here was a controller for a
complex view that includes an edit field, which is also important to note.</p>

<p>So, now all I needed was to open and close the <code>NSPopover</code>!  The simple
version of the code in <code>BRStatusItemPopoverController</code> looked like this:</p>

<div class="CodeRay">
  <div class="code"><pre>- (void)close {
  [_popover close];
  _shown = NO;
}

- (void)open {
  BRStatusItemIconView* view = _statusItem.view;
  [_popover showRelativeToRect:view.bounds ofView:view
    preferredEdge:NSMaxYEdge];
  _shown = YES;
}

- (void)activated:(BRStatusItemIconView*)sender {
  if (_shown) {
    [self close];
  } else {
    [self open];
  }
}</pre></div>
</div>


<p>Hit run to try it out, and&hellip; it works!  However, if the popover&rsquo;s content
view contained an <code>NSTextField</code>, you&rsquo;d find that the field refuses to
become first responder!  Even making sure the field was marked editable in
Interface Builder didn&rsquo;t help.  #FAIL.</p>

<h2>Fixing Edit Fields</h2>

<p>According to <a href="http://stackoverflow.com/questions/7214273/nstextfield-on-nspopover">this</a>
StackOverflow question, the problem with the <code>NSTextField</code> is the parent
window&rsquo;s inability to become the key window.  The answer in that question
is to write a global category for your application:</p>

<div class="CodeRay">
  <div class="code"><pre>NSWindow+canBecomeKeyWindow.h
@interface NSWindow (canBecomeKeyWindow)

@end

NSWindow+canBecomeKeyWindow.m
@implementation NSWindow (canBecomeKeyWindow)

//This is to fix a bug with 10.7 where an NSPopover with a text field
//cannot be edited if its parent window won't become key
//The pragma statements disable the corresponding warning for
//overriding an already-implemented method
#pragma clang diagnostic push
#pragma clang diagnostic ignored &quot;-Wobjc-protocol-method-implementation&quot;
- (BOOL)canBecomeKeyWindow
{
    return YES;
}
#pragma clang diagnostic pop

@end</pre></div>
</div>


<p>Trying this out appears to work when you click the status item.  You can
click the <code>NSTextField</code> and it becomes editable!  But, this has created a
subtle problem: the status item only works correctly if the application
that owns it is active.  When inactive, the status item weirdly requires a
double click to activate it, and even then, the double click sends two
<code>mouseDown:</code> events to <code>BRStatusItemIconView</code>, causing the interaction to
be out of phase with the user&rsquo;s intention.</p>

<h2>Fixing the double click</h2>

<p>Long story short, we need to control when the <code>NSStatusBarWindow</code> window,
the parent of <code>BRStatusItemIconView</code>, is allowed to become key.  When the
popover is opened, we want that to happen so that the popover&rsquo;s content
view can become first responder.  When the popover is closed, we want to go
back to the original behavior so that input to <code>BRStatusItemIconView</code>
doesn&rsquo;t get blocked.  But, we aren&rsquo;t familiar with the original behavior
because it&rsquo;s hidden by the original implementation of
<code>canBecomeKeyWindow:</code>.  Using a technique called <a href="http://cocoadev.com/?MethodSwizzling">method swizzling</a> that helps us do this.  Note that
there is an updated version of method swizzling discussed <a href="http://kevin.sb.org/2006/12/30/method-swizzling-reimplemented/">here</a>, but I&rsquo;ve
not implemented it in my code yet.  The new code looks like this:</p>

<div class="CodeRay">
  <div class="code"><pre>#import &quot;NSWindow_canBecomeKeyWindow.h&quot;
#include

BOOL shouldBecomeKeyWindow;
NSWindow* windowToOverride;

@implementation NSWindow (canBecomeKeyWindow)

//This is to fix a bug with 10.7 where an NSPopover with a text field
//cannot be edited if its parent window won't become key
//The pragma statements disable the corresponding warning for
//overriding an already-implemented method
#pragma clang diagnostic push
#pragma clang diagnostic ignored &quot;-Wobjc-protocol-method-implementation&quot;
- (BOOL)popoverCanBecomeKeyWindow
{
    if (self == windowToOverride) {
        return shouldBecomeKeyWindow;
    } else {
        return [self popoverCanBecomeKeyWindow];
    }
}

+ (void)load
{
    method_exchangeImplementations(
      class_getInstanceMethod(self, @selector(canBecomeKeyWindow)),
      class_getInstanceMethod(self, @selector(popoverCanBecomeKeyWindow)));
}
#pragma clang diagnostic pop

@end</pre></div>
</div>


<p>Now, <code>shouldBecomeKeyWindow</code> and <code>windowToOverride</code> just need to be set
correctly.  As I said above, we want <code>windowToOverride</code> to be the
<code>NSStatusBarWindow</code> and <code>shouldBecomeKeyWindow</code> to change based on the
status of the popover being visible.  The new
<code>BRStatusItemPopoverController</code> code is this:</p>

<div class="CodeRay">
  <div class="code"><pre>- (void)close
{
    [_popover close];
    shouldBecomeKeyWindow = NO;
    _shown = NO;
}

- (void)open
{
    BRStatusItemIconView* view = (BRStatusItemIconView*)_statusItem.view;

    shouldBecomeKeyWindow = YES;
    [_popover showRelativeToRect:view.bounds ofView:view
      preferredEdge:NSMaxYEdge];

    windowToOverride = view.window;
    [view.window becomeKeyWindow];
    _shown = YES;
}</pre></div>
</div>


<h2>Adding transient behavior</h2>

<p>Finally!  Things work as they should.  However, one cool thing about
<code>NSPopover</code>&rsquo;s default functionality is its ability to be a transient
presence for the user.  For applications that use the status bar, this is
good behavior.  However, because <code>NSPopoverBehaviorApplicationDefined</code> was
used, we have to duplicate that functionality.  Basically, we want the
popover to be able to disappear when the application resigns being active.
 This can be done by listening to the appropriate notifications in
<code>BRStatusItemPopoverController</code> and changing the ability of the
<code>NSStatusBarWindow</code> to become key, or close the popover.</p>

<div class="CodeRay">
  <div class="code"><pre>- (void)applicationDidResignActive:(NSNotification*)n
{
    if ((self.behavior == BRStatusItemPopoverBehaviorTransient) &amp;&amp;
       _shown) {
        [self close];
    } else if (self.behavior == BRStatusItemPopoverBehaviorPermanent) {
        shouldBecomeKeyWindow = NO;
    }
}</pre></div>
</div>


<h2>Conclusion</h2>

<p>Apple has some bugs to fix related to <code>NSPopover</code> and <code>NSStatusBarWindow</code>.
 It seems obvious to me that applications that live in the status bar
should use <code>NSPopover</code> to present complex user interfaces, but the
complexity of doing so right now is too high.</p>
