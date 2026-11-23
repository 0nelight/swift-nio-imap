# iOS Mail shows "not downloaded" for multipart/mixed with an empty text/plain part

## Symptom

A message consisting of an empty `text/plain` part followed by attachments opens on
iOS Mail with the grey italic placeholder *"This message has not been downloaded from
the server."* (German UI: *"Diese Nachricht wurde nicht vom Server geladen."*). The
attachments are not shown, and tapping to load does not change anything. The same
message displays correctly in other clients.

Seen on iOS 27.2.1 (24B156), `com.apple.email.maild` 3901.200.71.2.2, against
Dovecot 2.4.5.

## What the client does on the wire

[trace.txt](trace.txt): the client fetches only `RFC822.SIZE BODYSTRUCTURE` and
then nothing for that message, including in a session tagged `"event"
"user-initiated"`. Over 8 sessions it never requested `BODY[1]`, `BODY[2]` or
`BODY[3]`. The server answers every request immediately and correctly:

```
(("text" "plain" ("charset" "UTF-8" "format" "flowed") NIL NIL "7bit" 0 0 NIL NIL NIL NIL)
 ("application" "pdf" (...) NIL NIL "base64" 659660 NIL ("attachment" (...)) NIL NIL)
 ("application" "pdf" (...) NIL NIL "base64" 15038794 NIL ("attachment" (...)) NIL NIL)
 "mixed" ("boundary" "...") NIL NIL NIL)
```

The text part is 0 octets / 0 lines. That is the only unusual property of this message.

## Not this package's parser

`GrammarParser+Body.swift` accepts this BODYSTRUCTURE (`"7bit"` matches the
case-insensitive `parseFixedString`, `0 0` are valid numbers, the extension fields
follow the grammar). If parsing had failed, 27.2.1 would have downloaded the whole
message instead (`Failed to parse bodystructure … Falling back to downloading complete
message`, `Message` 24B156 @ `0x1badce85c`). That did not happen. The test suite has
no BODYSTRUCTURE with a zero-length part, though.

## Where it goes wrong: Message.framework / MIME.framework

The following comes from Ghidra decompilation of iOS 26.6.1 (23G83, iPhone14,6). The
behaviour on 27.2.1 on the wire is identical.

**1. A zero-length part is "complete" without a fetch.**
`-[MFMimePart _ensureBodyDataToOffset:resultOffset:downloadIfNecessary:isComplete:decoded:]`
(MIME @ `0x1d69a352c`) clamps the requested length to the part's range. For a
0-octet part, the network branch (`if (cached < requested)`) is skipped, and:

```c
_objc_msgSend_range(self);                 // part length = 0
if (partLength <= requested) requested = partLength;
...
if (requested == 0) {
    data = [MFMutableData new];            // empty
    goto done;                             // isComplete = YES
}
```

So the client decides locally that the body is fully loaded and empty, and it never
sends `BODY[1]`, which matches the trace.

**2. The message view loads in default-alternative mode.**
`-[MFMessageContentRequest requestHTMLRepresentationWithCompletionHandler:]` calls
`-[MFMessageLoadingContext load:0 … shouldDownload:YES]` (Message @ `0x1b286f31c`).
With mode `0`, `_reallyLoad:shouldDownload:` uses `defaultAlternativeInPart:` and skips
its in-loop "best alternative loaded empty document" check (see the device log section).

**3. A single document with `resultOffset == 0` is replaced by empty content, without an error.**
After the loop, `_reallyLoad:shouldDownload:` (Message @ `0x1b287d518`) calls
`__contentLoadedEmptyDocument` (`0x1b287ec3c`):

```c
if (count == 1 && [first isKindOfClass:MFWebMessageDocument])
    return resultOffset == 0 ? YES : [first isVisuallyEmpty];
return NO;
```

`resultOffset` counts only text bytes, so a 0-octet text part makes this true no matter
what the document contains. On true, the content is set to an empty constant,
`hasLoadedSomeContent = YES`, no error is set, and `hasLoadedCompleteBody` is set from
MIME's `isComplete` (YES, from step 1).

**4. No content and no error → the placeholder.**
`-[MFMessageContentRequest _processContentLoadingContextEvent:]` (Message @
`0x1b286f448`). The "appended content" flag is set only by the HTML/string append block
(`block_invoke_3` @ `0x1b2871388`, unconditionally, even for an empty string). Attachment
items (block @ `0x1b28715cc`) never set it.

```c
if (!appendedAnyContent) {
    if (event.error)                                         // 0x801 "Error loading content"
        ...;
    else if ([event hasLoadedBestAlternativePart] && [event hasLoadedCompleteBody])
        log("Did not find any content for message: %{public}@, using placeholder html");
}
```

**5. The placeholder text is `NO_BODY`.**
`-[MessageContentRepresentationRequestDelegate contentObjectID:placeholderHTMLForEmptyContentWithCompletionHandler:]`
(MobileMailUI @ `0x219505918`) calls `localizedStringForKey:@"NO_BODY" value:@"" table:@"Main"`
(disassembly at `0x219505950`) and formats it with `_MFErrorPageMarkupFormat` =
`<html dir=auto><body><i><font color=#888>%@</font></i></body></html>`. This is the only
use of `NO_BODY` in MobileMailUI. The MobileMail app uses its own `NO_BODY` only for the
mailbox-list preview.

So the string reads "not downloaded", but the code path is "loaded, and empty". The
`MESSAGE_PARTIALLY_DOWNLOADED` banner (`MFHasMoreContentBannerView`) is a different
path and is not involved.

## Device log (iOS 27.2.1)

From a sysdiagnose taken on the affected device right after opening the message
(`maild`, subsystem `com.apple.email`, category `MessageBodyLoader`; pointers and
database IDs replaced):

```
21:49:12.702  copySummaryForMessage start (downloadIfNecessary YES): <MFLibraryMessage …> … received:2026-11-09 14:55:45 +0000 … remoteID:99911
21:49:13.572  Error getting loading context event: NSError Domain=EMErrorDomain Code=2050
21:49:13.572  copySummaryForMessage end (downloadIfNecessary YES), no body summary loaded: <MFLibraryMessage …> … remoteID:99911
```

The same start → error 2050 → "no body summary loaded" sequence repeats 14 times between
21:49:12 and 21:50:20. Each time downloading is allowed, and each attempt ends within
25–40 ms, without any request reaching the server (see the trace).

`2050` is `0x802`. In `-[MFMessageLoadingContext _reallyLoad:shouldDownload:]` (23G83) that
error is created in four places. One of them sits in the loop's best-alternative mode
(mode ≥ 1): on the second pass, if `__contentLoadedEmptyDocument` is true, it logs

```
Best alternative part %{public}@ loaded empty document for message %{public}@.
Falling back to default alternative.
```

and sets `0x802`, which is never cleared. That fits the summary path here, if
`copySummaryForMessage` loads in best-alternative mode (not verified). It does **not**
produce the message-view placeholder: a request that carries an error takes the
`0x801` "Error loading content" branch in step 4, not the placeholder. The placeholder
comes from the post-loop branch in mode `0` (step 3), which sets no error.

The archive contains no `dsc/` shared-cache strings, so most `com.apple.email` lines,
including the placeholder line, could not be resolved to text.

## Not fully resolved

- Why the attachments disappear as well. With the full tree, `decodeMultipart` (MIME)
  returns `[WebDoc, MFAttachment, MFAttachment]`. That is three items, so step 3 would not
  fire and the attachments would render. Step 3 needs exactly one `MFWebMessageDocument`,
  so the part that is decoded cannot have attachment children: either the empty text
  part itself is selected, or the stored MIME tree lacks the PDF parts.
  The Swift section planner on 27.2.1, `BodyStructure.partsForHTMLBody(missingParts:additional:)`
  (`Message` 24B156 @ `0x1bb279abc`), was not traced down to its size-0 branch. It is only
  observed that it requests zero sections.
- The Objective-C side was decompiled from 23G83. The available 24B156 extract of
  `Message` lacks Objective-C stub fixups.

## Reproduction

[repro.eml](repro.eml): `multipart/mixed` with an empty `text/plain` and a small PDF
attachment, with the same structure as the affected message. **Not yet confirmed on a
device.** The observed case had a 15 MB attachment. A newly appended message is the newest
in its mailbox and may be prefetched by a different code path, so a small repro might not
trigger the same branch.

Steps: append to a mailbox an iOS device syncs, open it, and compare against the same
message with one line of text in the `text/plain` part.

## Suggested fix

Treat a 0-octet `text/*` part as "no displayable body" rather than "empty body": skip it
when choosing the best alternative inside `multipart/mixed`, so the attachments are
rendered. And do not show the "not downloaded" string for content that was fully loaded
and is empty.

A server cannot work around this honestly. It would have to misreport the part size or
alter the stored message.
