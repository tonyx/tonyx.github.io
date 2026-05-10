# Security, Privacy, and GDPR

Event Sourcing presents a unique challenge for privacy regulations like GDPR, particularly the "Right to be Forgotten." Because the Event Store is an immutable ledger, you cannot simply execute a `DELETE` statement to remove someone's personal data.

## The Golden Rule of PII

The primary recommendation is simple: **Personally Identifiable Information (PII) should not be kept in the Event Store.** 

Whenever possible, keep PII in a separate, mutable store (like standard Identity tables) and only reference non-identifiable GUIDs in your immutable Event Store.

## Event Ghosting and Scrambling

In scenarios where sensitive information has inadvertently entered the event stream, Sharpino provides advanced GDPR functions to "ghost" or scramble the eventual private data.

Because the event stream must remain replayable to reach the current valid state, Sharpino supports replacing specific events with "ghosted" versions. 

**The critical requirement for safe replacement:** 
The replacement event must be chosen so that reprocessing the stream leaves the system behavior completely unaltered. When the existing stored events are applied to a state that is "ghosted", they must still return an `Ok` (No error). Likewise, the newly introduced ghosted events must also yield an `Ok`. This guarantees the integrity of the stream is preserved, the application does not crash during replay, and the sensitive PII is permanently removed from the ledger.
