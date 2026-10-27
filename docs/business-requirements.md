# Business requirements: practice payment bot

Status: draft for review  
Date: 27 October 2026 (Europe/Kyiv)  
Product: a private Telegram bot for one practitioner (a psychologist is the example) who schedules sessions in Google Calendar and receives session payments on a PrivatBank account.

This document is the version to build from once the open decisions in section 16 are confirmed. Where a decision is still open, the rest of the document follows the recommended default and marks it with an assumption id.

## 1. Purpose

The owner wants three answers without opening the bank app or the calendar:

1. How much money clients paid during a chosen period, for the whole practice.
2. How much one client paid during that period.
3. Which clients missed a payment.

The same weekly picture should arrive on its own every Monday, for the previous week.

Calendar titles and bank payment lines do not share an id. The client dictionary is the bridge: it records what the owner calls the client in Google Calendar, and which names on PrivatBank payments belong to that same client.

## 2. Owner and job

The only user in this version is the practice owner. The owner creates the dictionary, connects the calendar and the bank, asks for reports, and resolves payments the bot cannot place.

Clients do not use the bot. The bot does not message them.

A typical practice has a few dozen active clients, about one session a week each, one fee per client, and payments in hryvnia.

## 3. What this version includes

- One Telegram account, allowed to use the bot.
- One Google Calendar, read from Google.
- One PrivatBank account, read as a statement.
- A client dictionary the owner edits in the bot.
- On-demand reports for an arbitrary period, plus the four presets: this week, previous week, this month, previous month.
- A Monday report for the previous calendar week.
- Owner corrections: assign a payment, ignore a bank line, record cash, write off a remainder.

## 4. Assumptions used below

| Id | Assumption |
| --- | --- |
| A1 | One practice, one owner, one calendar, one PrivatBank account. |
| A2 | “Business name” means the payer name printed on the PrivatBank payment. A client may have several payer names (the client, a spouse, a parent, an employer company). |
| A3 | The event title, after normalization, equals the client’s calendar name. |
| A4 | A session that remains on the calendar is charged, including a no-show. A Google-cancelled event, or a title that contains the word “скасовано” or “cancelled”, is not charged. |
| A5 | Each client has one session fee in UAH. |
| A6 | The owner can record a cash payment by hand. Other banks are not connected. |
| A7 | Payments arrive on a PrivatBank business account (a ФОП IBAN), read through the PrivatBank business statement. A credit with no recognizable payer stays unmatched. |
| A8 | The Monday report runs at 09:00 Europe/Kyiv and contains the full weekly picture described in section 11. |
| A9 | The bot’s interface language is Ukrainian. Amounts use a space as the thousands separator and a comma as the decimal mark, with the currency shown as UAH. |
| A10 | A payment covers the oldest unpaid session of that client first. A payment that arrives after the allowed window still clears that debt. The session stays visible as paid late. |
| A11 | Weeks run Monday 00:00 through Sunday 23:59, Europe/Kyiv. Months are calendar months in that zone. The owner’s “given period” is an inclusive pair of dates in that zone. |

## 5. Client dictionary

Each client is one row. The owner adds, edits, deactivates, and reactivates rows in the bot. Deactivation keeps history and stops new sessions of that name from entering the ledger. Rows are not deleted.

| Column | Required | What it holds |
| --- | --- | --- |
| Calendar name | Yes | The event title the owner uses for this client in Google Calendar. One primary name. |
| Extra calendar names | No | Further titles that mean the same client. |
| Payer names | Yes, at least one | Names to look for on a PrivatBank payment: counterparty name or payment purpose. This is the “business name” column, and it is a list. |
| Session fee | Yes | Amount in UAH the client is expected to pay for one session, down to the kopiyka. |
| Days allowed to pay | Yes | Calendar days after the session date. The default for a new client is 3. Zero means payment is due the same day. |
| Start date | Yes | Sessions and payments before this date are ignored for this client. The default is the day the row is created. The owner can move it earlier. |
| Status | Yes | Active or inactive. |
| Note | No | Free text for the owner. It is never parsed. |

The bot stores an internal id and the times of creation and last edit.

## 6. What counts as a session

The bot reads events from the connected calendar.

A billable session is an event that meets all of the following:

- The event has already started.
- Google’s event status is not cancelled.
- The normalized title does not contain “скасовано” or “cancelled” as a word.
- The normalized title equals the calendar name, or an extra calendar name, of exactly one active client.
- The event start is on or after that client’s start date.

Normalization for titles and names:

- Trim and collapse spaces.
- Compare without regard to case.
- Treat apostrophe characters (`'`, `’`, `ʼ`) as absent.
- Ignore one trailing period.

An event whose title matches two clients is not billed. It appears under “needs a decision” until the dictionary is unambiguous.

The calendar remains the source of sessions. Cancelling or deleting an event there removes that charge the next time a report runs. Each report is calculated again from the calendar, the bank, the dictionary, and the owner’s corrections.

## 7. What counts as a payment

### 7.1 Bank payments

The bot imports incoming credits on the connected account. For each credit it keeps the bank document id, accounting date, amount in UAH, counterparty name, and payment purpose.

Each bank document is imported once. A later refresh adds new documents and keeps documents already imported. A bank reversal arrives as its own later document; handling reversals and refunds is outside this version (see section 15).

Outgoing payments are outside the receipts total.

### 7.2 Manual payments

The owner can record cash (or a payment received outside this PrivatBank account): client, date, amount, note. Manual payments enter that client’s ledger and are labeled cash. They are included in “money received from clients” and kept out of the bank reconciliation.

### 7.3 Three buckets for every bank credit

Every incoming bank credit in a period sits in exactly one bucket:

| Bucket | Meaning |
| --- | --- |
| Attributed | Tied to one client by a payer-name match, or by an owner assignment. |
| Needs a decision | No client matched, or more than one client matched, and the owner has not assigned it. |
| Ignored | The owner marked it as not session income (for example a transfer from the owner’s own account). |

Check that must always hold:

`attributed bank credits + needs a decision + ignored = all incoming credits in the period`

## 8. How a payment is tied to a client

An unassigned credit is attributed automatically when exactly one active client matches.

A client matches when any of their payer names matches the counterparty name or the payment purpose, using the same normalization as titles, plus:

- A leading legal-form word is removed from both sides before comparison: ТОВ, ФОП, ПП, АТ, ПАТ, ПрАТ.
- The dictionary phrase matches a whole phrase: word boundaries on both sides, words in the same order.
- “Іван” does not match “Іваненко”. “Промінь” matches “ТОВ Промінь”.

The owner copies payer names from real payments. The bot does not treat “Олена Коваль” and “КОВАЛЬ ОЛЕНА ІВАНІВНА” as the same string. Both can be stored on the client when both appear.

An owner assignment overrides the automatic match. Editing the dictionary recalculates automatic matches and leaves owner assignments in place.

## 9. How a session becomes paid, waiting, or overdue

Each client has a ledger from their start date. Sessions and payments are applied in chronological order. On the same date, payments are applied before sessions.

Rules:

- A payment is applied to the oldest session of that client that still has an unpaid remainder.
- Any leftover amount stays as credit and applies to the next sessions.
- A session’s remainder is its fee minus payments applied to it, minus any write-off.
- Remainder of zero means the session is paid.
- A positive remainder, while the allowed days have not ended, means waiting.
- A positive remainder, after the allowed days have ended, means overdue. This is “missed a payment”.
- The allowed window ends at 23:59 Europe/Kyiv on `session date + allowed days`.
- A later payment still clears an overdue session. The session is then paid late.
- A new payment always reduces the oldest remainder first, even when a newer session is the one the owner had in mind.

Statuses shown to the owner:

| Status | When |
| --- | --- |
| Paid | Remainder is zero, and the payment that closed it fell on or before the window end. |
| Paid late | Remainder is zero, and that closing payment fell after the window end. |
| Waiting | Remainder is above zero, and the window has not ended. |
| Overdue | Remainder is above zero, and the window has ended. |

“Missed a payment” in the owner’s request is the overdue status. Waiting sessions are listed separately so a Monday report does not call a Friday session unpaid when the client still has days left to pay.

## 10. Reports the owner can ask for

Dates the owner types are inclusive, Europe/Kyiv. Presets follow assumption A11.

Every total can be opened down to the sessions and payments that produced it.

### 10.1 Money received, all clients

For the period, show:

- Headline: money received from clients. This is attributed bank credits whose accounting date is in the period, plus manual payments whose date is in the period.
- Bank subtotal, cash subtotal, count of payments, count of clients who paid.
- One line per client who received money in the period: calendar name, amount, payment count. Sorted by amount, largest first.
- Needs a decision: count and amount.
- Ignored: count and amount.
- The reconciliation check from section 7.3.

Clients with no payment in the period are omitted from the breakdown. The headline is cash received in the period. It is not “fees for sessions held in the period”. A payment on Monday for last Friday is counted in the new week.

### 10.2 Money received, one client

The owner picks a client by calendar name or payer name. When more than one row fits, the bot asks which one.

For that client and period, show:

- The same received total, limited to this client.
- Each payment: date, amount, source (bank or cash), payer name, purpose shortened for the chat.
- Each billable session in the period: date and time, fee, status, remainder.
- Credit still unapplied, if any.

### 10.3 Clients who missed a payment

Default view: every overdue session of every active client, as of the moment the owner asks.

For each overdue session: calendar name, session date and time, fee, remainder, days overdue. Grouped by client, with a client subtotal. Oldest session first.

The owner can limit this list to sessions that took place in a chosen period. Older overdue sessions are then summarized as one line: how many clients, how much remainder, so a weekly filter does not hide old debt.

Waiting sessions are a second, shorter list: client, session date, remainder, date the window ends.

When nothing is overdue, the bot says so.

### 10.4 Dictionary and setup

Before reports are meaningful, the owner:

1. Opens the bot from their own Telegram account. That account is the only one that can use it.
2. Connects Google and chooses the calendar.
3. Connects PrivatBank and chooses the account.
4. Adds clients.
5. Leaves the Monday report on, or sets another hour, or turns it off.

With an empty dictionary, a receipts report still runs: all credits fall into “needs a decision”, which lets the owner check the bank connection.

## 11. Monday report

When the schedule is on, the bot sends one report every Monday at the configured time (default 09:00 Europe/Kyiv) to the owner’s Telegram chat.

The period is the previous calendar week, Monday through Sunday, Europe/Kyiv.

The message contains, in this order:

1. The period.
2. Money received from clients, then the per-client lines.
3. Overdue sessions whose session date falls in that week.
4. One summary line for older overdue remainders still open.
5. Waiting sessions from that week whose window is still open, with the date each window ends.
6. Needs a decision: count and amount.
7. The reconciliation line.

A week with no sessions and no payments still produces a report that says the week was empty.

If the calendar or the bank cannot be read, the bot retries three times during that morning. After the retries fail, it sends a failure message and does not send figures. A partial total is not presented as the weekly result.

The owner can run this same report on demand for the previous week, or for another period using the reports in section 10.

The owner can turn the schedule off or change the hour. The hour is on the hour, Europe/Kyiv.

## 12. Corrections

The owner can:

- Assign a “needs a decision” credit to one client, or move an attributed credit to another client.
- Mark a credit as ignored, and undo that.
- Record a manual payment, and void it.
- Write off some or all of a session remainder, with a note.

Corrections are stored with the time and the previous value. Recalculation uses them.

The owner can export the dictionary, the ledger, and the corrections, and can delete all stored practice data.

## 13. Access, privacy, and money safety

- Messages from any other Telegram account receive no practice data and cannot change settings.
- Google access is read-only.
- Bank access can read the statement and cannot create or send a payment.
- Tokens are stored encrypted. The bot never sends a token, a full account number, or a card number into the chat.
- Stored practice data is limited to what matching and reports need: dictionary fields, event id, event title, event start, event status, bank document id, accounting date, amount, counterparty name, purpose, and corrections.
- Event descriptions, locations, and attendee lists are not stored.
- Session notes and anything about the content of therapy are out of the product.
- Reports in Telegram contain client names and amounts. The owner accepts that this chat history holds that data.
- The practice owner is the controller of the clients’ personal data. This version is a tool for that one owner.
- Ledger history is kept for as long as the practice data exists, because a later payment has to find the oldest unpaid session. There is no automatic deletion of old sessions.

## 14. Acceptance scenario

As of Tuesday 27 October 2026, 09:00 Europe/Kyiv. Report period: Monday 19 October through Sunday 25 October 2026.

Dictionary:

| Calendar name | Payer names | Fee | Days allowed | Start |
| --- | --- | --- | --- | --- |
| Олена Коваль | Коваль Олена | 1 500,00 | 2 | 1 Oct 2026 |
| Андрій Савчук | ТОВ Промінь | 2 000,00 | 5 | 1 Oct 2026 |
| Ірина Мельник | Мельник Ірина | 1 500,00 | 3 | 1 Oct 2026 |

Facts:

| When | Fact |
| --- | --- |
| Mon 19 Oct | Session “Олена Коваль”. |
| Mon 19 Oct | Incoming 10 000,00 from the owner’s own account. Owner marks it ignored. |
| Wed 21 Oct | Incoming 1 500,00, counterparty “КОВАЛЬ ОЛЕНА”, purpose “консультація”. |
| Tue 20 Oct | Session “Андрій Савчук”. No payment. Window ended Sun 25 Oct. |
| Fri 23 Oct | Session “Ірина Мельник”. |
| Thu 22 Oct | Incoming 800,00, purpose “Переказ коштів”, no known payer. |
| Mon 26 Oct | Incoming 1 500,00, counterparty “МЕЛЬНИК ІРИНА”. This date is outside the report week. It closes Ірина’s Friday session on the last allowed day. |

Expected result for 19–25 October:

| Figure | Value |
| --- | --- |
| Money received from clients | 1 500,00 |
| Олена Коваль | 1 500,00, session 19 Oct paid |
| Андрій Савчук | 0 received. Session 20 Oct overdue, remainder 2 000,00 |
| Ірина Мельник | 0 received inside the week. Session 23 Oct paid on time by the 26 Oct payment |
| Needs a decision | 800,00 |
| Ignored | 10 000,00 |
| Incoming credits | 12 300,00 = 1 500 + 800 + 10 000 |

A second ledger check, separate from that week. Client fee 1 500, window 2 days:

- Session on 6 Oct, no payment. Overdue from 9 Oct.
- Session on 13 Oct.
- Payment of 1 500 on 13 Oct.

The 13 Oct payment closes the 6 Oct session (paid late). The 13 Oct session stays unpaid. Oldest debt is paid first.

## 15. Outside this version

- More than one owner, practitioner, calendar, or bank account.
- Banks other than PrivatBank, and currencies other than UAH.
- Messages, invoices, or payment links sent to clients.
- Tax calculations and tax-office documents.
- A discounted package whose price is different from the session fee times a number of sessions. Prepayment of whole session fees already works as ledger credit.
- Guessing that two different name spellings are the same person.
- Refunds and statement reversals.
- Editing the calendar from the bot.
- Storing a frozen copy of a past report. Asking again recalculates.

## 16. Decisions still open

The document above uses the recommendation in the third column. A correction to any row changes the build.

| # | Decision | Recommendation |
| --- | --- | --- |
| Q1 | Is this one owner’s practice, or a product where each practitioner connects their own calendar and bank? | One owner, one calendar, one PrivatBank account (A1). |
| Q2 | Does “business name” mean the name printed on the PrivatBank payment, and may one client have several payer names? | Yes. A list of payer names (A2). |
| Q3 | What do real event titles look like, and how is a cancellation or a no-show written? | Title equals the calendar name. Cancelled events and the words “скасовано” / “cancelled” are not charged. A no-show that stays on the calendar is charged (A3, A4). |
| Q4 | Is the fee the same for every session of a client? | One fee per client (A5). |
| Q5 | Do some clients pay cash or through another bank? | Cash is entered by hand. Other banks are not connected (A6). |
| Q6 | Does the money land on a PrivatBank ФОП IBAN, or on a personal card? | Business statement of one IBAN (A7). |
| Q7 | Monday at 09:00 Europe/Kyiv, with the full report in section 11? | Yes (A8). |
| Q8 | Ukrainian interface? | Yes (A9). |

The most useful sample, when it is convenient to share, is three or four real event titles and one real incoming payment line (counterparty and purpose), with any extra personal detail removed. That sample confirms Q2, Q3, and Q6.
