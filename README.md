![Logo for clipboard-polyfill: an icon of a clipboard fading into a drafting paper grid.](clipboard-polyfill-logo.svg)

# `clipboard-polyfill`

Make copying on the web as easy as:

    clipboard.writeText("hello world");

As of October 2017, this library is a polyfill for the modern `Promise`-based [asynchronous clipboard API](https://www.w3.org/TR/clipboard-apis/#async-clipboard-api).

## Why `clipboard-polyfill`?

Browsers have implemented several clipboard APIs over time, and writing to the clipboard without [triggering bugs in various old and current browsers](https://github.com/lgarron/clipboard-polyfill/blob/master/experiment/Conclusions.md) is fairly tricky. In every browser that supports copying to the clipboard in some way, `clipboard-polyfill` attempts to act as close as possible to the async clipboard API. (Read to the end of this document for all the limitations.)

Note: If you only need to copy text and want a super simple polyfill that gets you 80% of the way, consider using [this gist](https://gist.github.com/lgarron/d1dee380f4ed9d825ca7).

# Usage

Get the source using one of the following:

- Download [`build/clipboard-polyfill.js`](https://raw.githubusercontent.com/lgarron/clipboard-polyfill/master/build/clipboard-polyfill.js) and include it using a `<script>` tag.
- `npm install clipboard-polyfill` and import as `clipboard`.

## Plain Text

Copy text to the clipboard (all modern browsers):

    clipboard.writeText("This text is plain.");

Read text from the clipboard (IE 9-11 and Chrome 65+):

    clipboard.readText().then(console.log, console.error);

Caveats:

- Browsers may require a user gesture or user permission to access the clipboard. In particular, you should write text only in response to an event listener, e.g. a button click listener.
- Reading fails if the clipboard does not contain `text/plain` data.

## Other Data Types (e.g. HTML)

Write (all modern browsers):

    var dt = new clipboard.DT();
    dt.setData("text/plain", "Fallback markup text.");
    dt.setData("text/html", "<i>Markup</i> <b>text</b>.");
    clipboard.write(dt);

Read (IE 9-11, Chrome 65+):

    // The success callback receives a clipboard.DT object.
    clipboard.read().then(console.log, console.error);

Caveats:

- Currently, `text/plain` and `text/html` are the only data types that can be written to the clipboard across most browsers.
- Unsupported data types will be silently dropped. In general, it is not possible to tell which data types will be dropped.
- This part of the clipboard API is still under active discussion, and may change.
- Currently, reading will only return the `text/plain` data type, if it is on the clipboard.

## Interface

    clipboard {
      static write:     (data: clipboard.DT)  => Promise<void>
      static writeText: (s: string) => Promise<void>
      static read:      () => Promise<clipboard.DT>
      static readText:  () => Promise<string>
      static suppressWarnings: () => void
    }

    clipboard.DT {
      constructor()
      setData: (type: string, value: string): void
      getData: (type: string): string | undefined
    }

## A note on `clipboard.DT`

The asynchronous clipboard API works like this:

    var dt = new DataTransfer();
    dt.setData("text/plain", "plain text");
    navigator.clipboard.write(dt);

Ideally, `clipboard-polyfill` would take a `DataTransfer`, so that the code above works verbatim when you replace `navigator.clipboard` with `clipboard`. However, *the `DataTransfer` constructor cannot be called* in most browsers. Thus, this library uses a light-weight alternative to `DataTransfer`, exposed as `clipboard.DT`:

    var dt = new clipboard.DT();
    dt.setData("text/plain", "plain text");
    clipboard.write(dt);


## This is way too complicated!

Try [this gist](https://gist.github.com/lgarron/d1dee380f4ed9d825ca7) for a simpler solution.

## [Can I use](http://caniuse.com/#feat=clipboard) it?

- Chrome 42+
- Firefox 41+
- Opera 29+
- Internet Explorer 9+ (text only)
- Edge
- Desktop Safari 10+
- iOS Safari 10+ (text only)

`clipboard-polyfill` uses a variety of heuristics to get around compatibility bugs. Please [let us know](https://github.com/lgarron/clipboard-polyfill/issues/new) if you are running into compatibility issues with any of the browsers listed above.

### Limitations

- In Microsoft Edge, it seems to be impossible to detect whether the copy action actually succeeded ([Edge Bug #14110451](https://developer.microsoft.com/en-us/microsoft-edge/platform/issues/14110451/), [Edge Bug #14080262](https://developer.microsoft.com/en-us/microsoft-edge/platform/issues/14080262/)). `clipboard-polyfill` will always call `resolve()` in Edge.
- In Microsoft Edge, only the *last* data type you specify is copied to the clipboard ([Edge Bug #14080506](https://developer.microsoft.com/en-us/microsoft-edge/platform/issues/14080506/)).
  - `DataTransfer` and `clipbard.DT` keep track of the order in which you set items. If you care which data type Edge copies, call `setData()` with that data type last.
- On iOS Safari ([WebKit Bug #177715](https://bugs.webkit.org/show_bug.cgi?id=177715)) and Internet Explorer, only text copying works.
  - On iOS Safari, `clipboard-polyfill` needs to use the DOM to copy, so the text will be copied as rich text. `clipboard-polyfill` attempts to use shadow DOM in order to avoid some of the page formatting (e.g. background color) from affecting the copied text. However, such formatting might be copied if shadow DOM is not available.
  - In other browsers, writing copy data that does *not* include the `text/plain` data type will succeed, but also show a console warning:

> clipboard.write() was called without a `text/plain` data type. On some platforms, this may result in an empty clipboard. Call `clipboard.suppressWarnings()` to suppress this warning.

- `clipboard-polyfill` attemps to avoid changing the document selection or modifying the DOM. However, `clipboard-polyfill` will automatically fall back to using such techniques if needed:
  - On iOS Safari, the user's current selection will be cleared. This *should* not happen on other platforms unless there are unanticipated bugs. (Please [file an issue](https://github.com/lgarron/clipboard-polyfill/issues/new) if you observe this!)
  - On iOS Safari and under certain conditions on desktop Safari ([WebKit Bug #177715](https://bugs.webkit.org/show_bug.cgi?id=156529)), `clipbard-polyfill` needs to add a temporary element to the DOM. This will trigger a [mutation observer](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver) if you have attached one to `document.body`. Please [file an issue](https://github.com/lgarron/clipboard-polyfill/issues/new) if you'd like to discuss how to detect temporary elements added by `clipboard-polyfill`.
- `read()` currently only works in Internet Explorer.
  - Internet Explorer can only read `text/plain` values from the clipboard.
- Internet Explorer does not have a native `Promise` implementation, so the standalone build file for `clipboard-polyfill` also includes `stefanpenner`'s [`es6-promise` polyfill](https://github.com/stefanpenner/es6-promise). This adds significant size to the build. Please [file an issue](https://github.com/lgarron/clipboard-polyfill/issues/new) if you're interested in a minimal build without Internet Explorer support.
- Microsoft Edge (at least EdgeHTML version <17) does not write `text/html` to the clipboard using the Windows `CF_HTML` clipboard format ([Edge Bug #14372529](https://developer.microsoft.com/en-us/microsoft-edge/platform/issues/14372529/)), which prevents other programs (including other browsers) from recognizing the copied HTML data ([issue #73](https://github.com/lgarron/clipboard-polyfill/issues/73)). `clipboard-polyfill` currently does not attempt to work around this issue.
