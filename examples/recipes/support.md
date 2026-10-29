<!-- Generated from the packaged MCP guide; do not edit this copy. -->

# Product support: ask Rupert

Ask Rupert, the product-support assistant, any question about Entity Enricher itself, continue the thread you opened, and read the support team's follow-ups.

## When to ask

Rupert knows the product: features, tools and settings, what an `error_code`, a limit or a warning means, why a result or a schema looks the way it does, what a plan includes. Ask when the tool descriptions and these guides do not answer, and before telling the user that something is impossible or guessing at a cause. Rupert does not run anything on the user's data and never sees the organization's records: describe the situation in the message instead — the tool you called, the `error_code` or message you got, the schema, job or record concerned (ids are fine, secrets never).

Every question counts against the organization's daily support quota (a plan limit; the free plan allows a few dozen a day). Each `ask_support` result says `questions_remaining` (`null` = unlimited), and the call fails with `support_question_limit_reached` once it is spent. Ask one well-stated question rather than a series of fragments, and do not ask the same question twice.

## One thread per topic

`ask_support` without `conversation_id` opens a thread and returns its id. Pass that id to the next question on the same topic: Rupert keeps the context of the exchange, and the support team reads one coherent thread. A new topic is a new thread. `list_support_conversations` finds the id of a thread opened earlier, in this session or another; `get_support_conversation` reads a whole thread.

The call blocks until the answer is complete, up to about two minutes, and returns the full reply. `support_unavailable` means the relay failed before answering; the question still counts and the thread exists — read it later rather than asking again at once.

## The support team's follow-ups

A member of the support team may answer in the thread after Rupert did. Two things make that answer reach the user:

- The result of `ask_support` on a thread holding a follow-up you have not read carries it as `new_admin_messages`; `threads_with_unread_admin_reply` lists your other threads with one waiting. Relay such a follow-up to the user, and if it changes the question, quote it to Rupert yourself in your next message.
- Rupert never reads these follow-ups: its own session continues from the questions and answers alone. Reading a thread (`get_support_conversation`) marks it read for you, as asking does.

## Who can ask

A thread belongs to a person. A client signed in with OAuth or a personal API key can ask; an organization access key gets `support_requires_user`. `support_url`, when present, opens the thread on the Users page for an organization owner or a system admin. Attachments are ordinary uploads (`upload_attachment`): up to five per question, images (PNG, JPEG, WebP, GIF, at most 2 MB each, three per question) are shown to Rupert, documents go as their extracted text, anything else fails with `unsupported_support_attachment`. `locale` asks for the answer in one of the UI languages; the default is English.
