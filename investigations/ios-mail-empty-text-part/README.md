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

- Why the attachments disappear as well. What is established (23G83):
  - The `multipart/mixed` container is what gets decoded, not the text child.
    `-[MFMessageStore bestAlternativeForPart:]` (MIME @ `0x1d698f718`) descends only for
    `multipart/alternative`; `defaultAlternativeForPart:` calls it, and only the Exchange
    store overrides it.
  - A full tree would show the PDFs. `isAttachment` (`0x1d69a1d24`) is YES for
    `application/pdf`, and `decodeMultipart` (`0x1d69a7db4`) returns
    `[WebDoc, MFAttachment, MFAttachment]`. That is three items, so step 3 cannot fire.
  - A sparse EMLX would give the full tree. `IMAP2MIME.Message.makeSparse(messageHeaders:bodyStructure:lineEnding:)`
    (Message @ `0x1b2eb0560`) writes `--boundary` plus part headers for every
    BODYSTRUCTURE part. It, and "Filling out empty & missing section(s)" (`0x1b29d43a0`),
    are reached only from the completion of a section download (`0x1b29d2d94`). Here no
    section download runs: the text part needs no fetch, and the PDFs are deferred.
  - The other parse outcomes are excluded. In `__parseHeaders` (MIME @ `0x1d69a5210`),
    if `--boundary` is not found in the body data, no subparts are created. A non-empty
    body then becomes one synthesized `text/plain` child. An empty body gives zero
    children, which shows `CANNOT_DECODE_MULTIPART_MESSAGE` (an error document, not
    `NO_BODY`). Missing body data gives `0x802`.

  So by elimination (inferred): the body data that maild parses for this message is not
  built from BODYSTRUCTURE. It is non-empty but contains no `--boundary`, so it
  collapses to a single text child, giving one visually empty `MFWebMessageDocument` →
  step 3 → `NO_BODY`, with no attachments. A device log should show the fault-level line
  `Mime parsing: Couldn't find MIME boundary %@ in message…`.
- Where that data comes from. After sync, only `headers.emlx` exists: the message
  headers plus `X-Apple-Content-Length` (Message @ `0x1b27b0f2c`).
  `-[MFMailMessageLibrary bodyDataForMessage:…]` (`0x1b27ca084`) looks for `full.emlx`,
  then `partial.emlx`, then a database row. It never reads the body from `headers.emlx`.
  With none of those present, `-[MFLibraryStore bodyDataForMessage:…]` (`0x1b27913b8`)
  logs `#CacheLoads fetching body data from network` and calls
  `-[MFSwiftIMAPMailboxTaskManager fetchDataForMessage:partial:]` (`0x1b298b9a8`):
  `Starting synchronous download for message UID %u`. The result is split at the end of
  the header, and both halves (header and body) replace what the parser sees. A nil
  body would give `0x802`, so the result was non-nil. What the Swift download returns
  when it plans zero sections was not decoded. There are two possibilities:
  - **H1:** real headers with a non-empty body that lacks `--boundary` → one synthesized
    `text/plain` child.
  - **H2:** a header block without `Content-Type` (which defaults to `text/plain`) and an
    empty body.

  H1 is more likely, since H2 would require the Swift side to rebuild the headers.
  Confirmed so far: the download completes within 25–40 ms without any request (see the
  trace).

**The "complete when fetching, absent when stored" asymmetry (checked in Ghidra).**
The 0-octet part is treated as two different things by two layers, and that is what
strands the message:
- *Nothing to fetch.* `_ensureBodyDataToOffset:…` (MIME @ `0x1d69a352c`) clamps the
  requested length to 0 and returns empty data with `isComplete = YES`, without any
  network (step 1). The section enqueue path in `IMAP2Persistence` has a matching
  zero-count branch: `FUN_1b2d0b5e8` (23G83 @ `0x1b2d0b5e8`, the "Requested to download
  %ld sections" / "Adding %ld sections for message" function) tests the section-set count
  and, when it is 0, takes a branch that only inserts the message id into a set and adds
  no per-section download records. So no section data is ever enqueued or stored for this
  message.
- *Treated as not-yet-downloaded.* No stored section means no `full.emlx` / `partial.emlx`
  / DB row, so `bodyDataForMessage:` falls through to the synchronous-download fallback
  above. In other words, a part that the fetch planner calls "complete" never produces
  stored body data, and the store then calls the message "not downloaded". This matches
  the original guess that the bug is a `0` vs empty/`""` confusion — the same class of
  mistake as the zero-length literal framing bug, one layer up, between the fetch planner
  and the section store.

**Where the 0-octet part is dropped (27.2.1, confirmed).** Read with the Swift tooling from
`ghidra-swift` (`git@gcr-git:ghidra-swift.git`), on `Message` 24B156:

1. *What counts as missing.* `checkWithoutCacheForMissingParts(uids:)` (`0x1badcdeec`)
   builds a `MailboxPersistenceHelper.MissingMessageParts` per UID (`0x1badd8368`) from the
   stored `PartStatus` (`known([Part: SparsePartInfo.Status])`, `full`, `allMissing`,
   `inaccessible`). A body part is missing only if its `IMAP2MIME.SparsePartInfo.Status`
   is `.empty(length:)` (predicate `0x1badd8ff0`), an attachment only if it is
   `.emptyAttachment` (`0x1badd910c`). Type and case names from `ipsw swift-dump`.
2. *Which parts the body needs.* `BodyStructure.partsForHTMLBody(missingParts:additional:)`
   (`0x1bb279abc` → `0x1bb27a4b8`) picks the `text/*` parts (preferring `html` inside
   `multipart/alternative`) and keeps those listed in, or under, the missing parts
   (`Set.filter` `0x1bb27d0f8` with predicate `0x1bb27b60c`). It has no size check; for this
   message it yields part `1`.
3. *0-octet parts are removed from the request.* The section planner (`0x1badd9b84`, the
   function that logs `Will request sections {%s}, plan: %{public}s, …`) does:

   ```c
   planned  = FUN_1baddce8c(...);          // partsForHTMLBody ∩ missing, + attachments per DownloadPlan
   zero     = FUN_1bac91598(bodyStructure); // BodyStructure.enumerateParts: every single part
                                            // whose fields.octetCount == 0  (0x1bac91648)
   sections = planned.subtracting(zero);    // Set.subtracting @ 0x1badda4a8
   ```

   `FUN_1bac91648` skips multipart nodes and inserts a single part when the 8-byte field at
   offset `0x48` of its `BodyStructure.Fields` is 0. In `Fields` (this package,
   `Grammar/Body/Field/Fields.swift`) that is `octetCount` (`parameters` 0x18, `id`,
   `contentDescription`, `encoding` 0x10 each, then `octetCount`); `fields` sits at `+8` in
   `Singlepart` after `kind`. In the call at `0x1badda4a8`, `planned` is `self` (`x1`) and
   the zero set is `other` (`x0`).

So `{1} − {1} = {}`: with the default plan (attachments deferred), nothing is requested. That
is deliberate — a 0-octet part needs no bytes — but nothing records it as present either:
its status never becomes `.normal`, no sparse EMLX is written (that happens only when a
section download completes), and the message keeps looking undownloaded. Every later load
repeats the same plan, which is why the synchronous download finishes in milliseconds
without a request. This is the `0` vs "empty" confusion: the planner treats a 0-octet part
as already complete, the store treats it as not yet downloaded.

Still open: the exact bytes the synchronous download returns in this state (H1 vs H2
below), and the 24B156 equivalent of the zero-count branch in the 23G83 section enqueuer
(what, if anything, it marks for a message with zero sections).

**Distinguishing H1 from H2 on a device** (if a labelled build is available): append the
repro twice, once with the text part at 0 bytes and once at 1 byte, and compare these
lines in a sysdiagnose:
- `Fetched body data of length %lu from full data of length %lu …`
- `Couldn't find MIME boundary` (H1)
- `Updating range to %@ for part %@` on the top part without a boundary line (H2)
- `Will request sections {…}, plan: …` (the empty plan)
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
