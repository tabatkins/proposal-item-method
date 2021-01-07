# Proposal for an `.at()` method on all the built-in indexables

A TC39 proposal to add an .at() method to all the basic indexable classes (Array, String, TypedArray)

**Stage: 3**

**Champions: Tab Atkins, Shu-yu Guo**

**Proposed Spec Text:** <https://tc39.github.io/proposal-relative-indexing-method/>

ToC
----

1. [Rationale](#rationale)
	1. [Existing Methods](#existing-methods)
1. [Proposed Edits](#proposed-edits)
1. [Polyfill](#polyfill)
1. [Web Incompatibility History](#web-incompatibility-history)
	1. [DOM Justifications](#dom-justifications)
		1. [Convertable Interfaces](#convertable-interfaces)
    1. [Possible Issues](#possible-issues)
	    1. [Possible DOM Compat Issues](#possible-dom-compat-issues)

Rationale
---------

For many years, programmers have asked for the ability to do "negative indexing" of JS Arrays, like you can do with Python. That is, asking for the ability to write `arr[-1]` instead of `arr[arr.length-1]`, where negative numbers count backwards from the last element.

Unfortunately, JS's language design makes this impossible. The `[]` syntax is not specific to Arrays and Strings; it applies to all objects. Referring to a value by index, like `arr[1]`, actually just refers to the property of the object with the key "1", which is something that any object can have. So `arr[-1]` already "works" in today's code, but it returns the value of the "-1" property of the object, rather than returning an index counting back from the end.

There have been many attempts to work around this; the most recent is a restricted proposal to make it easier to access just the last element of an array (<https://github.com/tc39/proposal-array-last>) via a `.last` property.

This proposal instead adopts a more common approach, and suggests adding a `.at()` method to Array, String, and TypedArray, which takes an integer value and returns the item at that index, with the negative-number semantics as described above.

This not only solves the long-standing request in an easy way, but also happens to solve a separate issue for [various DOM APIs, described below](#dom-justifications).

### Existing Methods

Currently, to access a value from the end of an indexable object, the common practice is to write `arr[arr.length - N]`, where N is the Nth item from the end (starting at 1).  This requires naming the indexable twice, additionally adds 7 more characters for the `.length`, and is hostile to anonymous values; you can't use this technique to grab the last item of the return value of a function unless you first store it in a temp variable.

Another method that avoids some of those drawbacks, but has some performance drawbacks of its own, is `arr.slice(-N)[0]`. This avoids repeating the name, and thus is friendly to anonymous values as well. However, the spelling is a little weird, particularly the trailing `[0]` (since `.slice()` returns an Array). Also, a temporary array is created with all the contents of the source from the desired item to the end, only to be immediately thrown away after retrieving the first item.

Note, however, the fact that `.slice()` (and related methods like `.splice()`) already have the notion of negative indexes, and resolve them exactly as desired.

Possible Issues
---------------

`.at()` might also be web incompatible for reasons yet unknown.

Proposed Edits
--------------

<https://tc39.github.io/proposal-relative-indexing-method/>

Polyfill
--------

(Rough polyfill; correctly implements the behavior for well-behaved objects, but not guaranteed to match spec behavior precisely for edge cases, like calling the method on `undefined`.)

```js
function at(n) {
	// ToInteger() abstract op
	n = Math.trunc(n) || 0;
	// Allow negative indexing from the end
	if(n < 0) n += this.length;
	// OOB access is guaranteed to return undefined
	if(n < 0 || n >= this.length) return undefined;
	// Otherwise, this is just normal property access
	return this[n];
}

// Other TypedArray constructors omitted for brevity.
for (let C of [Array, String, Uint8Array]) {
    Object.defineProperty(C.prototype, "at",
                          { value: at,
                            writable: true,
                            enumerable: false,
                            configurable: true });
}
```

## Implementations

* Spec-compliant polyfills using the old name of `.item()`: [Array.prototype.item](https://www.npmjs.com/package/array.prototype.item), [String.prototype.item](https://www.npmjs.com/package/string.prototype.item)
* Browsers:
  * Firefox Nightly 86
  * [Safari Technology Preview 118](https://webkit.org/blog/11439/release-notes-for-safari-technology-preview-118/)

## Web Incompatibility History

The original iteration of this proposal proposed the name of the method to be `.item()`. Unfortunately, this was found out to be not web compatible. Libraries, notably YUI2 and YUI3, were duck-typing objects to be DOM collections based on the presence of a `.item` property. Please see [#28](https://github.com/tc39/proposal-relative-indexing-method/issues/28), [#31](https://github.com/tc39/proposal-relative-indexing-method/issues/31), and [#32](https://github.com/tc39/proposal-relative-indexing-method/issues/32) for more details.

Captured below is the original motivation for choosing the `.item()` name and the original concerns.

### DOM Justifications

A recent addition to the WebIDL spec is `ObservableArray<>` (thanks @domenic!), a proxy over an Array that allows web APIs to expose something that to page authors looks exactly like an Array, but still allows the browser to intercept get/set/delete/etc of indexed properties, enforcing type checks and other requirements exactly like they do today with named properties.

We plan to start using this for most APIs that want to expose a list of something, but we'd also like to, when possible, upgrade *older* APIs to use this as well; the fact that many older APIs use bespoke interfaces that badly and incompletely copy the Array interface is a consistent source of frustration for web authors.

(For example, `document.querySelectorAll()` returns, not an Array, but a NodeList, which supports indexed properties and `.length`, and so can be treated as an Array in basic ways, but has only a tiny selection of the Array prototype methods. Popular methods like `.map()` are missing, requiring authors to write code like `[...document.querySelectorAll("a")].map(foo)`.)

This upgrade can *almost* be done in-place, just swapping the various bespoke interfaces with ObservableArray, avoiding breaking anything that doesn't explicitly test the value's type.  There is *one* exception: all of them have a `.item()` method, which returns the value at the passed index.

(This is a remnant of the very old (1990s-era) belief that Java was a reasonable language to use on the web, and so APIs were designed in a "lowest common denominator" style for use in both JS and Java. Java didn't have the ability to use indexed properties at the time unless you were actually a Java array, so the .item() method was a compromise that worked identically in both languages.)

It's highly likely there is code that relies on using `.item()` on these interfaces, and we don't want to risk breakage there.

We *could* address this by subclassing ObservableArray and adding `.item()` in the subclass. However, that would mean the values aren't of type Array; various type-checking methods in the community looking for an Array would fail.

Or we could just add `.item()` to ObservableArray itself, as it's a proxy wrapper around Array. This would be confusing and weird however, making it appear that Array had such a method even tho it's not on the prototype.

The ideal solution for us, instead, is to add `.item()` to the Array prototype itself,
and for completeness/consistency, to the other indexable types that support the same general suite of index-related properties like `.slice()`.

As such, the name `.item()` is a requirement of this proposal; changing it to something else would still help authors, but would fail to satisfy the DOM needs.

#### Convertable Interfaces

Assuming this proposal is adopted, the following legacy interfaces should be upgradable into ObservableArray:

* [NodeList](https://dom.spec.whatwg.org/#nodelist)
* Possibly [DOMTokenList](https://dom.spec.whatwg.org/#domtokenlist) as a subclass
* [CSSRuleList](https://drafts.csswg.org/cssom/#cssrulelist)
* [StyleSheetList](https://drafts.csswg.org/cssom/#stylesheetlist)
* Possibly [CSSStyleDeclaration](https://drafts.csswg.org/cssom/#cssstyledeclaration) and [MediaList](https://drafts.csswg.org/cssom/#medialist), as subclasses
* [FileList](https://w3c.github.io/FileAPI/#dfn-filelist)

(maybe others, list is ongoing)

### Possible Issues

The obvious looming issue with this, as with any addition to the built-ins, is the possibility that the name `.item()` is already added to these classes' prototypes by a framework with an incompatible definition, and added using one of the fragile patterns that avoids clobbering built-in names, so that code depending on the framework's definition will then break when it's instead given the new built-in definition.

I'm prepared to eat my words, but I suspect that any library adding a `.item()` method to Array or the other indexables is going to be giving it compatible or identical semantics to what's outlined here; I can't imagine what else such a method name could possibly correspond to.

There's good evidence that we're probably safe here, tho: none of MooTools, Prototype, or Ext add `.item()` to Array; those are generally the most dangerous libraries for this kind of addition (see: [smooshgate](https://developers.google.com/web/updates/2018/03/smooshgate)), so if we're safe there it's much more likely we'll be safe in general.

### Possible DOM Compat Issues

The .item() method defined by all of the interfaces [listed above](#convertable-interfaces) has a common structure:

```webidl
	SomeType? item(unsigned long index);
```

If you follow the definition chain in WebIDL, you end up with a conversion algorithm for `unsigned long` into internal numbers that *mostly* matches what JS does for indexes in `.slice()`, with a few differences around the edges:

* WebIDL treats Infinity and -Infinity as 0, while JS leaves them as is.
* WebIDL modulos the value by 2^32 (using JS's "modulo" math op, so negatives become positive), while JS does not.
* If the index is out of range, WebIDL returns `null`, while JS returns `undefined`.

The first means that code that is accidentally passing infinities into `.item()` and relying on it returning the item at index 0 will break, as it will now get `undefined`. I find this very unlikely to be problematic.

The second means that code passing extremely large numbers that are *just a little bit* larger than 2^32 (so the modulo brings them back into a reasonable index) and relying on that to return something from the list will break, as it will now get `undefined`.  I also find this very unlikely to be problematic.

The second also means that code relying on small negative numbers being modulo'd into the vicinity of 4 billion, and thus returning `null`, will break, as it will now return items from near the end of the list.  I find this slightly likely, and believe we will need to do some instrumentation/testing to ensure it happens below our threshold for breakage.  (There is a small chance that such code is *expecting* to recieve a value from the end of the list and is currently broken, and will be fixed by this change.)

The third means that code which is testing for the presence of an item by *explicitly* comparing the return value with `null` will break, as it will now receive `undefined` and think a value was returned.  I also find this slightly likely. In my experience, however, most such code is written either as `== null` or simply uses the truthiness of the return value (since a valid index will always return an object, which is truthy); both of these kinds of tests will continue to work after the change.

---

We could potentially preview any of these changes before attempting to accept this proposal fully, so we know whether it's realistic to do such an upgrade, and thus whether the `.item()` name it a hard requirement or can be freely bikeshedded.

In particular, testing negative indexes would be fairly simple, just requiring a change to `signed long` and an extra line in the algorithms of the methods.

Testing returning `undefined` is also plausible; tho still slightly awkward to *express* in WebIDL (requiring the return type to be written as `any`), it's a tiny change to the algorithms of the methods.
