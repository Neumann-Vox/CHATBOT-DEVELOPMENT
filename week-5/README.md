# Week 5: Library Chatbot v2

**Author:** IHIMBAZWE Divinse · **GitHub:** Neumann-Vox · **Course:** Chatbots Development (AITCD001)

## Overview

In Assignment 4 the library chatbot gave every message one tag and one answer. Version 2 sends each intent to its own handler function, understands several requests in one message ("what time do you open and how much is printing"), and remembers the reader's name and the last intent answered. It also handles a side question in the middle of a room booking and understands a new greeting intent.

## Intents and handlers

| **Intent** | **Handler** | **Fixed reply, or worked out from the message?** |
| --- | --- | --- |
| name | handle_name | Worked out from the message: it finds the word after "my name is", stores it in `context` with a capital first letter, and replies with it. |
| rooms | handle_rooms | Worked out from the state of the booking: it starts the booking, or says "Your booking is already under way." if one is running. |
| hours | handle_hours | Fixed reply from RESPONSES. |
| borrow | handle_borrow | Fixed reply from RESPONSES. |
| renew | handle_renew | Fixed reply from RESPONSES. |
| fines | handle_fines | Worked out from the message: the number of days is read from the text; with no number, the fixed general reply. |
| printing | handle_printing | Fixed reply from RESPONSES. |
| greeting | handle_greeting | Fixed reply. It says the chatbot is an automated service, not a librarian. |

## How the fines handler works

It searches the message for a number followed by "day" or "days" (the pattern `LATE`). If it finds one, it turns that number into an integer, multiplies it by 200 RWF, and takes the smaller of that result and 5,000 RWF, so the fine never goes over the cap. It then replies "For N days late, the fine is X RWF per book." If the message names no number of days, it returns the general fines reply instead. For example, "3 days late" gives 600 RWF and "40 days late" gives 5,000 RWF, because 40 x 200 = 8,000 is capped.

## Several requests in one message

`split_requests` lowercases the message and cuts it at the connectors (and, but, also, then and a question mark). It tidies each piece with `normalise()` and drops the empty pieces. A comma is not a connector. `route` then finds one tag for each piece with `match()`, skips pieces that match nothing and skips an intent that has already been found, so every intent appears once, in the order asked. An overlap inside one request is settled by the rule table: the first rule that matches wins. For "can i renew a late book", renew is higher in the table than fines, so only the renewal reply is given.

`ASKS` holds **rooms**. Its reply, "Which day would you like the room?", is a question the reader must answer next. `respond` puts it at the end so the reader finishes on the question, and the other answers (such as the opening hours) are not buried after it.

## Memory

`context` is a dictionary with two entries for the whole conversation: `name` (the reader's name, or None) and `last_intent` (the last intent answered). Saying "bye" clears both `context` and the booking flow, and the goodbye uses the name if there is one.

The follow-up rule: when a message matches no intent but the last intent answered was fines and the message names a number of days, it is answered as a fines question. After any other answer it is not, and the chatbot gives the fallback.

During a booking the flow sees the message first. If the flow accepts it as an answer, the booking moves on. If it does not, the message is routed and answered, and the booking question is asked again, so the booking is kept.

## Test results

- Assignment 4 test set, one tag per message: 32 of 35

- Assignment 4 test set, routed: 33 of 35

Routing fixes `'how many books can i take and what are the fines'` (expected borrow+fines). Two messages still fail: `'rnw my bk'` (expected renew, got fallback) and `'can i extend my deadline'` (expected fallback, got renew).

## Test conversations

| **#** | **What it tests** | **Expected** | **What the chatbot did** | **Pass or fail** | **If it failed: cause and part responsible** |
| --- | --- | --- | --- | --- | --- |
| 1 | Two requests joined by "and" | Both answered, in the order asked: the opening hours, then the printing prices. | Gave the opening hours, then the printing prices. | Pass | |
| 2 | One request that matches two intents | The renewal reply only. "late" also matches fines, but renew is higher in the table, and the reader asked one thing. | Gave only the renewal reply. | Pass | |
| 3 | The question goes last | The opening hours first. The reply ends with the booking question (which day), because that is what the reader must answer next. | Gave the opening hours, then "Which day would you like the room? Monday to Saturday." | Pass | |
| 4 | Remembering a name | A greeting, the name acknowledged, the printing prices, and a goodbye that uses the name Grace. | Greeted the reader, said "Nice to meet you, Grace.", gave the printing prices, and said "Thank you, Grace. Murakoze." | Pass | |
| 5 | A follow-up with no intent of its own | The fine for 3 days (600 RWF), then the second message answered as a fines question: the fine for 10 days (2,000 RWF). | Replied "For 3 days late, the fine is 600 RWF per book." then "For 10 days late, the fine is 2,000 RWF per book." | Pass | |
| 6 | A side question during a booking | The hours are given and the time question asked again. The booking then completes with a confirmation for Friday afternoon, 4 people. | Gave the hours and asked "Morning, afternoon or evening?" again. The booking then ended with "A room on fri, afternoon, for 4 people. Is that correct?" | Pass (the day is shown as "fri", the way the provided booking flow stores it) | |
| 7 | Two questions with no connecting word | Both answered: the borrowing limit, then the fines. | Gave only the borrowing limit. The fines question was never answered. | Fail | There is no connecting word, so `split_requests` returned one piece and `match()` returned only the first rule that fits (borrow). Part responsible: the split (`CONNECTORS`). |
| 8 | An answer that is also a question | The evening slot is taken, and the closing time on Saturday is given as well. | Replied "How many people? Between 1 and 8." The closing time was not answered, and nothing was said about the evening slot. | Fail | The flow saw the message first, found "evening" and accepted it as the time, so the message was never routed and the closing time was lost. The chatbot also has no data about which slots are taken. Part responsible: the flow (and the order in `reply_to`, where the flow sees the message before the router). |
| 9 | A conversation that tests memory | **Expected before running:** "my name is eric" gives "Nice to meet you, Eric."; "i am 12 days late what is the fine" gives 2,400 RWF; "and for 30 days?" is answered as fines and capped at 5,000 RWF; "how much is printing" gives the printing prices; "and for 5 days?" is NOT a fines follow-up now (the last intent was printing), so it gets the fallback; "bye" says "Thank you, Eric." | Every reply was as expected: 2,400 RWF, then 5,000 RWF, then the printing prices, then the fallback for "and for 5 days?", then "Thank you, Eric. Murakoze." | Pass | |
| 10 | A conversation designed to break the chatbot | **Expected before running:** "what time do you open, how much is printing" answers only the opening hours (a comma is not a connector); "i want to cancel my room booking" wrongly starts a new booking ("room booking" is in the rooms rule); "never mind" keeps asking "Which day?" because the flow only accepts a day; "bye" is the only way out. | Everything happened as predicted: only the hours were given, a new booking was started, "never mind" got "Which day? Monday to Saturday.", and only "bye" ended it. | Fail (as designed) | Three causes. The comma problem is in the split (`CONNECTORS`). Starting a booking when the reader wants to cancel is the rule table (the rooms pattern matches "room booking"). Being unable to leave the booking is the flow (it has no cancel step). |

## Known limitations

- Conversation 7: two questions with no connecting word are treated as one request, so only the first intent is answered. Cause: the split.
- Conversation 8: an answer and a question in the same message inside a booking loses the question, and the chatbot cannot say that a slot is taken. Cause: the flow takes the message first and there is no availability data.
- Conversation 10, comma: "what time do you open, how much is printing" answers only the hours. Cause: a comma is not a connector, so the split does not cut it.
- Conversation 10, cancelling: "i want to cancel my room booking" starts a new booking. Cause: the rooms pattern matches "room booking" and the rule table has no cancel intent.
- Conversation 10, leaving a booking: "never mind" is treated as a failed answer, and the reader is stuck until they say "bye". Cause: the flow has no way to cancel.
- `'rnw my bk'` is not understood. Cause: no rule contains "rnw" or "bk".
- `'can i extend my deadline'` is answered as a renewal. Cause: "extend" is in the renew pattern, and the rule table matches words, not meaning.
- The booking confirmation shows "fri" instead of "Friday". Cause: the provided flow stores the first three letters of the day.

## Running this notebook

- Open IHIMBAZWE_Divinse_Assignment_5.ipynb in Google Colab.

- Run the cells in order. The chat cell waits for input, so do not use Run all.

## What was learned

I learned that a router of handler functions makes the chatbot easier to extend, because a new intent needs only a rule, a handler and one entry in the dictionary. I also learned that splitting a message at connectors lets one message hold several requests, but that anything the split or the rule table cannot see, such as two questions with no connecting word, still fails. Memory made the chatbot feel more natural, but the booking flow taking every message first shows that the order in which the parts run decides what the chatbot can understand.

