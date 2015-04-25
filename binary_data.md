The state of binary data in the browser
====

Or: "So you wanna store a Blob, huh?"
-----

### TLDR

Don't try to directly store Blobs in IndexedDB, unless you want to cry. Browsers still suck at it. 

[PouchDB](https://github.com/pouchdb/pouchdb) and [blob-util](https://github.com/nolanlawson/blob-util) have workarounds to avoid all the browser bugs.

### Long version

I know it's 2015, and Blobs/IndexedDB should be universally supported already. But sadly they're not, so here's the sorry state of things.

Browser have three ways of storing data: [LocalStorage](http://caniuse.com/#feat=namevalue-storage), [WebSQL](http://caniuse.com/#feat=sql-storage), and [IndexedDB](http://caniuse.com/#feat=indexeddb). They all suck for different reasons, which is why there are so many abstraction layers out there: PouchDB, YDN-DB, LocalForage, Lawnchair, MakeDrive, etc.

Browsers don't consistently handle Blobs either. The [caniuse.com page for Blobs](http://caniuse.com/#search=blob) is a bit disingenuous; really IE and Firefox should be yellowy-green, because they don't consistently support all the `canvas` and `FileReader` methods. Blobs in Chrome also have severe bugs before v43.

So let's see all the different browsers and storage engines, and how they stack up:

LocalStorage
----

Supported by [most browsers](http://caniuse.com/#feat=namevalue-storage), althought not Chrome extensions, Chrome apps, web workers, or service workers.

You can store Blobs in LocalStorage as base64 strings, which is really inefficient. Plus, many LocalStorage implementations only let you store up to 5MB, so you hit the limit pretty fast.

WebSQL and IndexedDB have [higher limits](http://www.html5rocks.com/en/tutorials/offline/quota-research/), so let's see how browsers work with those two.

Chrome
----

Supports both IndexedDB and WebSQL. Chrome originally got IndexedDB in v23.

**WebSQL** doesn't support storing Blobs themselves, only strings. You can store binary strings directly, which is the most efficient, but then [the `'\u0000'` character causes data to get lost](https://code.google.com/p/chromium/issues/detail?id=422690). PouchDB works around this by eliminating the `'\u0000'` in [a safe and very efficient way](https://github.com/pouchdb/pouchdb/pull/2900).

**IndexedDB** has many binary bugs in Chrome. Here's the history:

* **pre-v36:** Chrome didn't support Blobs at all, so you had to store data as base64-encoded strings. This includes Android up to Lollipop 5.0. ([Chromium issue](https://code.google.com/p/chromium/issues/detail?id=108012))
* **v37:** Chrome introduced broken support for Blobs ([issue](https://code.google.com/p/chromium/issues/detail?id=408120)). It was broken because the mimetype wasn't correctly returned.
* **v38:** The above bug was fixed in v38, but Chrome had two more Blob/IndexedDB bugs: [this one](https://code.google.com/p/chromium/issues/detail?id=447916) and [this one](https://code.google.com/p/chromium/issues/detail?id=447836). The second one in particular was a race condition causing data to be permanently unreadable, which was a big enough blocker that PouchDB continued downgrading Chrome to base64-only.
* **v43:** Chrome finally fixed all Blob bugs, so PouchDB auto-detects it and upgrades Chrome to Blobs ([test it out here](http://bl.ocks.org/nolanlawson/38e3cd6705f50b074566)).

Android
----

Android didn't support IndexedDB until 4.4 Kitkat, and as of this writing, [more than half of Android devices are still pre-Kitkat](https://developer.android.com/about/dashboards/index.html). Some Samsung/HTC Android 4.3 devices have a broken version of IndexedDB based on an older version of the spec. PouchDB detects this and falls back to WebSQL.

Additionally, many pre-4.4 devices don't support Blobs correctly - either they're using vendor prefixes like `window.webkitURL` or they use the deprecated `BlobBuilder` API. `blob-util` works around all these issues.

4.4 Kitkat devices will either have Chrome 30 or Chrome 33 depending on whether it's 4.4.0-4.4.1 or 4.4.2+. Lollipop is auto-updating; it debuted with Chrome v37 and is up to v42 as of this writing.

Safari/iOS
---

**WebSQL:** Safari WebSQL has [the same '\u0000' bug as Chrome](https://bugs.webkit.org/show_bug.cgi?id=137637) (on both iOS and desktop), as well as another bug that affects Safari pre-v7.1 and iOS pre-8.0 where all data is coerced to UTF-16 instead of UTF-8, meaning it takes up twice the space. PouchDB detects UTF-16 UTF-8 encoding and [reacts accordingly](https://github.com/pouchdb/pouchdb/pull/1733#issuecomment-38723096).

**IndexedDB:** The less said about Safari IndexedDB, the better. It [is so buggy](http://www.raymondcamden.com/2014/09/25/IndexedDB-on-iOS-8-Broken-Bad) that PouchDB, LocalForage, and YDN-DB all ignore it. For what it's worth, though, it doesn't support binary Blobs [according to HTML5Test.com](http://html5test.com/compare/browser/safari-8.0.html).

IE/Firefox
----

Neither one support WebSQL, but they're actually both great about supporting Blobs in IndexedDB. IE10 supports Blobs in IndexedDB without breaking a sweat, and Firefox has had them [since 2011](https://bugzilla.mozilla.org/show_bug.cgi?id=661877).

That being said, these two have bugs related to the Blob/FileReader APIs themselves:

IE doesn't have `FileReader.prototype.readAsBinaryString` (only `readAsArrayBuffer`), so if you want to convert a Blob to a binary string or a base64 string most efficiently, you want to use `readAsBinaryString` everywhere but IE.

Firefox, conversely, doesn't have the `canvas.toBlob` method, so if you want to convert a `canvas` to a Blob, you need to use `canvas.toDataURL` and convert the dataURL to a Blob instead. `blob-util` does this all under the hood.

More resources
---

A lot of this is documented in [the PouchDB FAQs](http://pouchdb.com/faq.html#data_types), [the PouchDB 3.0.6 release notes](http://pouchdb.com/2014/09/22/3.0.6.html), and ["10 things I learned from reading and writing the PouchDB source"](http://pouchdb.com/2014/10/26/10-things-i-learned-from-reading-and-writing-the-pouchdb-source.html).  More research on browser storage can be found [in this gist](https://gist.github.com/janl/d8efa4e404072037f7e0).

I'm not aware of any database library that stores Blobs as efficiently or in as many browsers as PouchDB (if I'm wrong, though, then let me know in the comments :)). You can even use [the localstorage adapter](http://pouchdb.com/adapters.html#pouchdb_in_the_browser) to store Blobs that way (in which case they will be inefficiently base64-encoded). And the proof is in the pudding: [the PouchDB test suite is insane](https://travis-ci.org/pouchdb/pouchdb/).