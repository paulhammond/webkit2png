#!/usr/bin/python

# webkit2png - makes screenshots of web pages
# http://www.paulhammond.org/webkit2png

__version__ = "0.8-dev"

# Copyright (c) 2004-2014 Paul Hammond
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in
# all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
# THE SOFTWARE.
#

import sys
import optparse
import re
import os

try:
    import Foundation
    import WebKit
    import AppKit
    import Quartz
    import objc
except ImportError:
    print "Cannot find pyobjc library files.  Are you sure it is installed?"
    sys.exit()


class AppDelegate(Foundation.NSObject):
    # what happens when the app starts up
    def applicationDidFinishLaunching_(self, aNotification):
        webview = aNotification.object().windows()[0].contentView()
        webview.frameLoadDelegate().getURL(webview)
        self.performSelector_withObject_afterDelay_("timeout:", None, self.timeout)

    def timeout_(self, obj):
        Foundation.NSLog("timed out!")
        AppKit.NSApplication.sharedApplication().terminate_(None)


class Webkit2PngScriptBridge(Foundation.NSObject):
    def init(self):
        self = super(Webkit2PngScriptBridge, self).init()
        self.is_stopped = False
        self.start_callback = False
        return self

    def stop(self):
        self.is_stopped = True

    def start(self):
        self.is_stopped = False
        self.start_callback()

    def isSelectorExcludedFromWebScript_(self, sel):
        if sel in ['stop', 'start']:
            return False
        else:
            return True


class WebkitLoad (Foundation.NSObject, WebKit.protocols.WebFrameLoadDelegate):

    # what happens if something goes wrong while loading
    def webView_didFailLoadWithError_forFrame_(self, webview, error, frame):
        if error.code() == Foundation.NSURLErrorCancelled:
            return
        print " ... something went wrong: "+error.localizedDescription()
        self.getURL(webview)

    def webView_didFailProvisionalLoadWithError_forFrame_(self, webview, error, frame):
        if error.code() == Foundation.NSURLErrorCancelled:
            return
        print " ... something went wrong: "+error.localizedDescription()
        self.getURL(webview)

    def makeFilename(self, URL, options):
        # make the filename
        if options.filename:
            filename = options.filename
        elif options.md5:
            try:
                import md5
            except ImportError:
                print "--md5 requires python md5 library"
                AppKit.NSApplication.sharedApplication().terminate_(None)
            filename = md5.new(URL).hexdigest()
        else:
            filename = re.sub('^https?', '', URL)
            filename = re.sub('\W', '', filename)
        if options.datestamp:
            import time
            now = time.strftime("%Y%m%d")
            filename = now + "-" + filename
        dir = os.path.abspath(os.path.expanduser(options.dir))
        if not os.path.exists(options.dir):
            os.makedirs(dir)
        return os.path.join(dir, filename)

    def saveImages(self, bitmapdata, filename, options):
        # save the fullsize png
        if options.fullsize:
            bitmapdata.representationUsingType_properties_(AppKit.NSPNGFileType, None).writeToFile_atomically_(filename + "-full.png", objc.YES)

        if options.thumb or options.clipped:
            # work out how big the thumbnail is
            width = bitmapdata.pixelsWide()
            height = bitmapdata.pixelsHigh()
            thumbWidth = (width * options.scale)
            thumbHeight = (height * options.scale)

            # make the thumbnails in a scratch image
            scratch = AppKit.NSImage.alloc().initWithSize_(Foundation.NSMakeSize(thumbWidth, thumbHeight))
            scratch.lockFocus()
            AppKit.NSGraphicsContext.currentContext().setImageInterpolation_(AppKit.NSImageInterpolationHigh)
            thumbRect = Foundation.NSMakeRect(0.0, 0.0, thumbWidth, thumbHeight)
            clipRect = Foundation.NSMakeRect(0.0, thumbHeight-options.clipheight, options.clipwidth, options.clipheight)
            bitmapdata.drawInRect_(thumbRect)
            thumbOutput = AppKit.NSBitmapImageRep.alloc().initWithFocusedViewRect_(thumbRect)
            clipOutput = AppKit.NSBitmapImageRep.alloc().initWithFocusedViewRect_(clipRect)
            scratch.unlockFocus()

            # save the thumbnails as pngs
            if options.thumb:
                thumbOutput.representationUsingType_properties_(AppKit.NSPNGFileType, None).writeToFile_atomically_(filename + "-thumb.png", objc.YES)
            if options.clipped:
                clipOutput.representationUsingType_properties_(AppKit.NSPNGFileType, None).writeToFile_atomically_(filename + "-clipped.png", objc.YES)

    def getURL(self, webview):
        if self.urls:
            if self.urls[0] == '-':
                url = sys.stdin.readline().rstrip()
                if not url:
                    AppKit.NSApplication.sharedApplication().terminate_(None)
            else:
                url = self.urls.pop(0)
        else:
            AppKit.NSApplication.sharedApplication().terminate_(None)

        nsurl = Foundation.NSURL.URLWithString_(url)
        if not (nsurl and nsurl.scheme()):
                nsurl = Foundation.NSURL.alloc().initFileURLWithPath_(url)
        nsurl = nsurl.absoluteURL()

        if self.options.ignore_ssl_check:
            Foundation.NSURLRequest.setAllowsAnyHTTPSCertificate_forHost_(objc.YES, nsurl.host())

        print "Fetching", nsurl, "..."
        self.resetWebview(webview)
        scriptobject = webview.windowScriptObject()
        scriptobject.setValue_forKey_(Webkit2PngScriptBridge.alloc().init(), 'webkit2png')

        webview.mainFrame().loadRequest_(Foundation.NSURLRequest.requestWithURL_(nsurl))
        if not webview.mainFrame().provisionalDataSource():
            print " ... not a proper url?"
            self.getURL(webview)

    def resetWebview(self, webview):
        rect = Foundation.NSMakeRect(0, 0, self.options.initWidth, self.options.initHeight)
        window = webview.window()
        window.setContentSize_((self.options.initWidth, self.options.initHeight))

        if self.options.transparent:
            window.setOpaque_(objc.NO)
            window.setBackgroundColor_(AppKit.NSColor.clearColor())
            webview.setDrawsBackground_(objc.NO)

        webview.setFrame_(rect)

    def captureView(self, view):
        bounds = view.bounds()
        if bounds.size.height > self.options.UNSAFE_max_height:
            print >> sys.stderr, "Error: page height greater than %s, clipping to avoid crashing windowserver." % self.options.UNSAFE_max_height
            bounds.size.height = self.options.UNSAFE_max_height
        if bounds.size.width > self.options.UNSAFE_max_width:
            print >> sys.stderr, "Error: page width greater than %s, clipping to avoid crashing windowserver." % self.options.UNSAFE_max_width
            bounds.size.width = self.options.UNSAFE_max_width

        view.window().display()
        view.window().setContentSize_(Foundation.NSSize(self.options.initWidth, self.options.initHeight))
        view.setFrame_(bounds)

        if hasattr(view, "bitmapImageRepForCachingDisplayInRect_"):
            bitmapdata = view.bitmapImageRepForCachingDisplayInRect_(bounds)
            view.cacheDisplayInRect_toBitmapImageRep_(bounds, bitmapdata)
        else:
            view.lockFocus()
            bitmapdata = AppKit.NSBitmapImageRep.alloc()
            bitmapdata.initWithFocusedViewRect_(bounds)
            view.unlockFocus()
        return bitmapdata

    # what happens when the page has finished loading
    def webView_didFinishLoadForFrame_(self, webview, frame):
        # don't care about subframes
        if (frame == webview.mainFrame()):
            scriptobject = webview.windowScriptObject()

            if self.options.js:
                scriptobject.evaluateWebScript_(self.options.js)

            bridge = scriptobject.valueForKey_('webkit2png')

            def doGrab():
                Foundation.NSTimer.scheduledTimerWithTimeInterval_target_selector_userInfo_repeats_(self.options.delay, self, self.doGrab, webview, False)

            if bridge.is_stopped:
                bridge.start_callback = doGrab
            else:
                doGrab()

    def doGrab(self, timer):
        webview = timer.userInfo()
        frame = webview.mainFrame()
        view = frame.frameView().documentView()

        URL = webview.mainFrame().dataSource().initialRequest().URL().absoluteString()
        filename = self.makeFilename(URL, self.options)

        bitmapdata = self.captureView(view)

        if self.options.selector:
            doc = frame.DOMDocument()
            el = doc.querySelector_(self.options.selector)

            if not el:
                print " ... no element matching %s found?" % self.options.selector
                self.getURL(webview)
                return

            left, top = 0, 0
            parent = el
            while parent:
                left += parent.offsetLeft()
                top += parent.offsetTop()
                parent = parent.offsetParent()

            zoom = self.options.zoom

            cropRect = view.window().convertRectToBacking_(Foundation.NSMakeRect(zoom * left, zoom * top, zoom * el.offsetWidth(), zoom * el.offsetHeight()))

            cropped = Quartz.CGImageCreateWithImageInRect(bitmapdata.CGImage(), cropRect)
            bitmapdata = AppKit.NSBitmapImageRep.alloc().initWithCGImage_(cropped)
            Quartz.CGImageRelease(cropped)

        self.saveImages(bitmapdata, filename, self.options)

        print " ... done"
        self.getURL(webview)


def main():

    # parse the command line
    usage = """%prog [options] [http://example.net/ ...]

Examples:
%prog http://google.com/            # screengrab google
%prog -W 1000 -H 1000 http://google.com/ # bigger screengrab of google
%prog -T http://google.com/         # just the thumbnail screengrab
%prog -TF http://google.com/        # just thumbnail and fullsize grab
%prog -o foo http://google.com/     # save images as "foo-thumb.png" etc
%prog -                             # screengrab urls from stdin
%prog /path/to/file.html            # screengrab local html file
%prog -h | less                     # full documentation"""

    cmdparser = optparse.OptionParser(usage, version=("webkit2png " + __version__))
    # TODO: add quiet/verbose options
    cmdparser.add_option("--debug", action="store_true",
                         help=optparse.SUPPRESS_HELP)

    # warning: setting these too high can crash your window server
    cmdparser.add_option("--UNSAFE-max-height", type="int", default=30000,
                         help=optparse.SUPPRESS_HELP)
    cmdparser.add_option("--UNSAFE-max-width", type="int", default=30000,
                         help=optparse.SUPPRESS_HELP)

    group = optparse.OptionGroup(cmdparser, "Network Options")
    group.add_option("--timeout", type="float", default=60.0,
                     help="page load timeout (default: 60)")
    group.add_option("--user-agent", type="string", default=False,
                     help="set user agent header")
    group.add_option("--ignore-ssl-check", action="store_true", default=False,
                     help="ignore SSL Certificate name mismatches")
    cmdparser.add_option_group(group)

    group = optparse.OptionGroup(cmdparser, "Browser Window Options")
    group.add_option(
        "-W", "--width", type="float", default=800.0,
        help="initial (and minimum) width of browser (default: 800)")
    group.add_option(
        "-H", "--height", type="float", default=600.0,
        help="initial (and minimum) height of browser (default: 600)")
    group.add_option(
        "-z", "--zoom", type="float", default=1.0,
        help='zoom level of browser, equivalent to "Zoom In" and "Zoom Out" in "View" menu (default: 1.0)')
    group.add_option(
        "--selector", type="string",
        help="CSS selector for a single element to capture (first matching element will be used)")
    cmdparser.add_option_group(group)

    group = optparse.OptionGroup(cmdparser, "Output size options")
    group.add_option(
        "-F", "--fullsize", action="store_true",
        help="only create fullsize screenshot")
    group.add_option(
        "-T", "--thumb", action="store_true",
        help="only create thumbnail sreenshot")
    group.add_option(
        "-C", "--clipped", action="store_true",
        help="only create clipped thumbnail screenshot")
    group.add_option(
        "--clipwidth", type="float", default=200.0,
        help="width of clipped thumbnail (default: 200)",
        metavar="WIDTH")
    group.add_option(
        "--clipheight", type="float", default=150.0,
        help="height of clipped thumbnail (default: 150)",
        metavar="HEIGHT")
    group.add_option(
        "-s", "--scale", type="float", default=0.25,
        help="scale factor for thumbnails (default: 0.25)")
    cmdparser.add_option_group(group)

    group = optparse.OptionGroup(cmdparser, "Output filename options")
    group.add_option(
        "-D", "--dir", type="string", default="./",
        help="directory to place images into")
    group.add_option(
        "-o", "--filename", type="string", default="",
        metavar="NAME", help="save images as NAME-full.png,NAME-thumb.png etc")
    group.add_option(
        "-m", "--md5", action="store_true",
        help="use md5 hash for filename (like del.icio.us)")
    group.add_option(
        "-d", "--datestamp", action="store_true",
        help="include date in filename")
    cmdparser.add_option_group(group)

    group = optparse.OptionGroup(cmdparser, "Web page functionality")
    group.add_option(
        "--delay", type="float", default=0,
        help="delay between page load finishing and screenshot")
    group.add_option(
        "--js", type="string", default=None,
        help="JavaScript to execute when the window finishes loading"
        "(example: --js='document.bgColor=\"red\";'). "
        "If you need to wait for asynchronous code to finish before "
        "capturing the screenshot, call webkit2png.stop() before the "
        "async code runs, then webkit2png.start() to capture the image.")
    group.add_option(
        "--noimages", action="store_true",
        help=optparse.SUPPRESS_HELP)
    group.add_option(
        "--no-images", action="store_true",
        help="don't load images")
    group.add_option(
        "--nojs", action="store_true",
        help=optparse.SUPPRESS_HELP)
    group.add_option(
        "--no-js", action="store_true",
        help="disable JavaScript support")
    group.add_option(
        "--transparent", action="store_true",
        help="render output on a transparent background (requires a web "
        "page with a transparent background)", default=False)
    cmdparser.add_option_group(group)

    (options, args) = cmdparser.parse_args()
    if len(args) == 0:
        cmdparser.print_usage()
        return
    if options.filename:
        if len(args) != 1 or args[0] == "-":
            print "--filename option requires exactly one url"
            return

    # deprecated options
    if options.nojs:
        print >> sys.stderr, 'Warning: --nojs will be removed in webkit2png 1.0. Please use --no-js.'
        options.no_js = True
    if options.noimages:
        print >> sys.stderr, 'Warning: --noimages will be removed in webkit2png 1.0. Please use --no-images.'
        options.no_images = True

    if options.scale == 0:
        cmdparser.error("scale cannot be zero")
    # make sure we're outputing something
    if not (options.fullsize or options.thumb or options.clipped):
        options.fullsize = True
        options.thumb = True
        options.clipped = True
    # work out the initial size of the browser window
    #  (this might need to be larger so clipped image is right size)
    options.initWidth = (options.clipwidth / options.scale)
    options.initHeight = (options.clipheight / options.scale)
    options.width *= options.zoom
    if options.width > options.initWidth:
        options.initWidth = options.width
    if options.height > options.initHeight:
        options.initHeight = options.height

    # Hide the dock icon (needs to run before NSApplication.sharedApplication)
    AppKit.NSBundle.mainBundle().infoDictionary()['LSBackgroundOnly'] = '1'

    app = AppKit.NSApplication.sharedApplication()

    # create an app delegate
    delegate = AppDelegate.alloc().init()
    delegate.timeout = options.timeout
    AppKit.NSApp().setDelegate_(delegate)

    # create a window
    rect = Foundation.NSMakeRect(0, 0, 100, 100)
    win = AppKit.NSWindow.alloc()
    win.initWithContentRect_styleMask_backing_defer_(rect, AppKit.NSBorderlessWindowMask, 2, 0)
    if options.debug:
        win.orderFrontRegardless()
    # create a webview object
    webview = WebKit.WebView.alloc()
    webview.initWithFrame_(rect)
    # turn off scrolling so the content is actually x wide and not x-15
    webview.mainFrame().frameView().setAllowsScrolling_(objc.NO)

    if options.user_agent:
        webview.setCustomUserAgent_(options.user_agent)
    else:
        webkit_version = Foundation.NSBundle.bundleForClass_(WebKit.WebView).objectForInfoDictionaryKey_(WebKit.kCFBundleVersionKey)[1:]
        webview.setApplicationNameForUserAgent_("Like-Version/6.0 Safari/%s webkit2png/%s" % (webkit_version, __version__))
    webview.setPreferencesIdentifier_('webkit2png')
    webview.preferences().setLoadsImagesAutomatically_(not options.no_images)
    webview.preferences().setJavaScriptEnabled_(not options.no_js)

    if options.zoom != 1.0:
        webview._setZoomMultiplier_isTextOnly_(options.zoom, False)

    # add the webview to the window
    win.setContentView_(webview)

    # create a LoadDelegate
    loaddelegate = WebkitLoad.alloc().init()
    loaddelegate.options = options
    loaddelegate.urls = args
    webview.setFrameLoadDelegate_(loaddelegate)

    app.run()

if __name__ == '__main__':
    main()
