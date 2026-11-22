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

**2. An empty best alternative is treated as an empty message.**
`-[MFMessageLoadingContext _reallyLoad:shouldDownload:]` (Message, around `0x1b287d77c`)
loads `bestAlternativeInPart:`. `__contentLoadedEmptyDocument` (`0x1b287ec3c`) returns
true for a single `MFWebMessageDocument` with `resultOffset == 0`, which logs

```
Best alternative part %{public}@ loaded empty document for message %{public}@.
Falling back to default alternative.
```

and if that is empty too, sets empty content with `hasLoadedCompleteBody = isComplete`
(YES, from step 1).

**3. Empty and complete → the "not downloaded" placeholder.**
`-[MFMessageContentRequest _processContentLoadingContextEvent:]` (Message @
`0x1b286f448`):

```c
if (!appendedAnyContent && event.error == nil &&
    [event hasLoadedBestAlternativePart] && [event hasLoadedCompleteBody])
    log("Did not find any content for message: %{public}@, using placeholder html");
```

**4. The placeholder text.**
`-[MessageContentRepresentationRequestDelegate contentObjectID:placeholderHTMLForEmptyContentWithCompletionHandler:]`
(MobileMailUI @ `0x219505918`) loads a key from the MobileMail `Main.strings` and wraps
it in `<i><font color=#888>…</font></i>`. The key is presumably `NO_BODY`, matching the
text shown.

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
is the error code created in four places, one of them directly after the
`Best alternative part … loaded empty document … Falling back to default alternative.`
log line. The log alone does not identify which of the four fired. Together with the
empty part and the absent fetch, it is consistent with the empty-document path above.

The archive contains no `dsc/` shared-cache strings, so most `com.apple.email` lines,
including the placeholder line, could not be resolved to text.

## Not fully resolved

- Why the attachments disappear as well. On the MIME side, `decodeMultipart` would
  still emit attachment placeholders for `multipart/mixed`. The likely cause is that
  the empty text part, rather than the container, is selected as the best alternative.
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
