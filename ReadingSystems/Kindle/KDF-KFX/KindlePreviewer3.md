# Kindle Format 10 (KFX)

Inquiring further into Kindle Previewer 3 (beta), we may get a hint of the conversion process to KFX.

At first sight, the process is epic. Please note [some folks have been retro-engineering already](http://www.mobileread.com/forums/showthread.php?t=263902). I can now confirm their findings. 

I’ll quote from and refer to that thread’s posts since it would be disrespectful to steal and paraphrase.

**TL;DR:** we’re screwed.

## Acknowledgments 

Thanks to jhowell, and all the others folks in the MobileRead thread, whose research proved so valuable when inquiring into dat shit.

## KFX Architecture

It seems KFX is a follow-on AZK, the Kindle format for iOS you’ll get when using Kindle Previewer 2.94.

More technical details about AZK [here](http://www.mobileread.com/forums/showpost.php?p=3097967&postcount=8) and [there](http://www.mobileread.com/forums/showpost.php?p=3100761&postcount=11).

As a reminder, AZK is **not** the file Kindle for iOS uses, it is indeed converted to KCR (Kindle Cloud Reader) when you sideload it on your iDevice. [Kindle Cloud Reader for iOS has been partly documented](https://github.com/FriendsOfEpub/WillThatBeOverriden/tree/master/ReadingSystems/Kindle/Kindle-iOS) and implies a lot of HTML + CSS sanitization.

So, to sum things up, like AZK (and KCR), **KFX is a binary version of JSON** (JavaScript Object Notation). In other words, it is somehow likely the new Kindle renderer is sharing common traits with Kindle Cloud Reader, i.e. JavaScript built on top of jQuery and making use of webviews. That is still unclear though, so correct me if I’m wrong.

Strictly speaking, KindleGen is no longer involved at this AZK/KFX conversion level. And there is no publicly distributed converter for those two formats. `EpubToKFXConverter-1.0.jar` is bundled with KindlePreviewer (like `azkcreator` for AZK) so you can probably make use of your command-prompt-fu to access it (see [this post in MobileRead forums](http://www.mobileread.com/forums/showpost.php?p=3262219&postcount=338)).

### History

It looks like KFX has been around [since 2013](http://www.mobileread.com/forums/showpost.php?p=3182980&postcount=167). But it was [intended for magazines at first](http://www.mobileread.com/forums/showpost.php?p=3184542&postcount=170). In other words, KFX has been repurposed.

### Technical details

- Like KF8, KFX is using a webkit-based rendering engine.
- Contents (excepted images) are encrypted/ofbuscated (AES-256) for both DRM-protected **and** -free files.
- Contents are divided into small JSON files (like AZK and KCR, you can [go check online](https://www.amazon.com/cloudreader) for the latter).
- Grayscale images are using an high compression format called JPEG-XR (transparency is not supported at the moment).
- Color images are delivered in JPEG (at least in iOS).
- It **seems** specific versions are prepared and delivered depending on device (e.g. grayscale images for eInk Readers).
- Hyphens are turned on/off using `-webkit-hyphens`.
- The hyphenation engine is [not written in JavaScript](http://www.mobileread.com/forums/showpost.php?p=3206602&postcount=237).

### Goals

In two words: Enhanced Typography.

KFX brings support for drop caps, hyphenation & justification (H&J), kerning, ligatures, etc.

What they don’t advertise though, are clever features that will enhance the user experience:

- the renderer takes user settings into account and will adapt drop caps, H&J and `float` images dynamically once a critical font size is reached;
- the renderer supports gaiji images (`inline` images);
- the renderer computes an WCAG2-compliant contrast ratio: `color` for text on background.

As the previous renderer (for KF8) [performed very poorly with kerning and ligatures](http://www.mobileread.com/forums/showpost.php?p=3172282&postcount=420), it could explain why they decided to build a new one and manage a lot of stuff when processing files so that the new renderer doesn’t have to. 

While this is **pure speculation**, another goal may have been the creation of a stronger walled garden. Indeed, it looks like it would be pretty useless to convert KFX to ePub (more about that later).

### Notes

KFX is usually not mentioned in the release notes of Kindle updates. The addition of Bookerly seems to be a hint, though.

## Kindle Previewer

[Kindle Previewer 3](http://www.amazon.com/gp/feature.html/?docId=1003018611) (beta) outputs KDF files, which is basically KFX data in an sqlite3 database instead of a KFX container. That could be an intermediate format like AZK—only this time they didn’t bother or have time to implement a bridge inside apps which support KFX.

There is a lot of interesting stuff to be found in KP3’s package: 

- `EpubToKFXConverter-1.0.jar`, the **KDF** converter;
- `mobicontentdumper`, which dumps the mobi files generated by KindleGen as json files;
- `yjhtmlcleanerapp`, which cleans up code that goes in the way;
- `coreprocessor.js`, a script (3100+ lines beautified) which aim is to parse CSS styles and alter HTML files.

That hints at a very complex process.

And indeed, that process is mind-blowing.

When converting a file with Kindle Previewer, which takes some time (yeah, that’s `coreprocessor` in action), temporary files are created. To sum things up:

- the link to the external stylesheets is erased (`head`);
- book styles are parsed and inlined to each tag (`style` attribute);
- computed styles are then inlined as well (`computedstyle` attribute).

You end up with something like this: 

```
<p class="noindent" style="margin-top: 0px; margin-bottom: 0px; text-align: justify; display: block; text-indent: 0%; margin-left: 0%; margin-right: 0%; " computedstyle="font_style:normal;font_weight:normal;font_variant:normal;width:496px;height:96px;" >Text</p>
```

Check file `enhanced.xhtml` for further details.

Computed styles are always the same: 

1. `font_style`;
2. `font_weight`;
3. `font_variant`;
4. `width`;
5. `height`.

The purpose of such an “enhancement” is currently unknown. The most probable assumption is that “they are determining the final styling result of css for each html element and then translating that to their own simplified binary document model” ([source](http://www.mobileread.com/forums/showpost.php?p=3262351&postcount=342)).

Indeed, “Html and css files are replaced by text with associated formatting instructions in binary data structures. The possible formatting instructions are based loosely on css properties, but changed and somewhat simplified.”

As regards drop caps, Amzn has created its own non-standard style properties which are being applied after several conditions have been met (e.g. it is not a floating image, `span` is the first element in `p`, paragraph is the first element of the parent, `font-size` of the `span` is bigger than `font-size` of `p`, it is not a raised cap, etc.). As a matter of fact, that’s quite epic in itself.

And now to the **ion** data representation…

I don’t really want to make it obscure so if you want further details, check [this post](http://www.mobileread.com/forums/showpost.php?p=3269649&postcount=360).

You can also check `ionnames.csv` ([link](https://github.com/FriendsOfEpub/WillThatBeOverriden/blob/master/ReadingSystems/Kindle/KDF-KFX/ionnames.csv)). You’ll see `epub:type` and CSS properties + values in there so maybe we could build a list of supported styles from here.

To sum things up, content is divided into fragments, and I quote jhowell: 

1. `document_data`, a list of sections in reading order;
2. each section is an HTML file in the source file. It has a page template and a reference to a story;
3. a story contains a list of content, content types being based on HTML e.g. `container` for nested `div`, `text` for `div`, `p`, `h1`, etc.;
4. formatting instructions are grouped into a `style` based on HTML attributes and CSS properties.

**Note:** if the parser encounters any unsupported style, it aborts conversion to KDF. A list of currently unsupported styles would be difficult to create if it doesn’t exist already, somewhere in KP3.

## KFX to ePub

You’re screwed. 

1. KFX is encrypted and there is no way to break it at the moment.
2. HTML semantic markup is lost (`text` for several tags, same for `list`, same for inline elements like `em`, etc.).
3. “Structure” relies heavily on styles and not semantic tags—and there goes your basic a11y.

## What’s next?

Well, Amazon is very secretive about KFX and is unlikely to document it, if not providing a public converter you can use to create files in this format. After all, in [Kindle Previewer 3 FAQ](http://www.amazon.com/gp/feature.html/?docId=1003018611), they write “You can’t side load your books with Enhanced Typesetting.” 

That may be temporary but since KFX is a work in progress (WIP), it seems reasonable to imagine they’ll manage that on their side for a while—it is a lot easier to update your own server and reprocess files when needed than to manage tens of thousands of people using a local/native converter to upload KFX files directly.

So, given this status of WIP, the whole process which turns documenting and maintaining default styles + overrides into a bloody nightmare, and the dynamic sanitization/styling which probably happens at the renderer level, **we won’t inquire further.** Sorry Not Sorry, life is short and I don’t want to waste any more time on this one.

You could try public shaming them to get documentation but I’m not convinced it would have any effect—remember AZK has not been documented either and it’s been years since its release.

If you want to sacrifice yourself though, please let us know. We’ll be glad to help kickstart your research where we left ours.
