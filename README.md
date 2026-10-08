# Assignment 4 — Rule-Based Chatbot for the Campus Library

**Student:** IHIMBAZWE DIVINSE  
**Course:** Chatbots Development (AITCD001)  
**Week:** 4  
**Assignment:** 4 — Rule-Based Chatbot for the Campus Library

## Overview

This project implements the supplied campus-library rule-based chatbot. The work covers the six given intents, a multi-turn study-room booking flow, testing, failure diagnosis, and documentation.

No new domain intents were designed. The implementation follows the specification supplied in the assignment.

## Part A — Pattern ordering

The rules are checked in this order:

1. `rooms`
2. `hours`
3. `borrow`
4. `renew`
5. `fines`
6. `printing`

The **rooms** rule is deliberately first. A request such as `i want to book a study room` contains the word `book`, which is also associated with borrowing. Checking `rooms` first prevents the more specific study-room request from being handled as a borrowing request.

All intent patterns are wrapped in `\b` word boundaries. The classifier is case-insensitive.

## Intent responses

| Intent | Response |
|---|---|
| hours | We are open 08:00 to 20:00 Monday to Friday, and 09:00 to 13:00 on Saturday. Closed on Sunday. |
| borrow | You may borrow up to 4 books at a time, for 14 days. |
| renew | A loan can be renewed twice online, unless another reader has reserved the book. |
| fines | Overdue books are 200 RWF per book per day, capped at 5,000 RWF. |
| rooms | Starts the multi-turn study-room booking flow. |
| printing | Printing is 50 RWF per page in black and white, 200 RWF in colour. |

## Out-of-scope requests

The chatbot does not answer questions outside its six supported areas. These return the fallback response:

> I can help with library opening hours, borrowing, renewals, overdue fines, study-room bookings, and printing/photocopying. Please speak to a librarian if you need help with something else.

Examples include:

- `can you write my assignment`
- `do you sell coffee`
- `what is the wifi password`

## Part B — Booking flow

The study-room flow collects three values one at a time.

### Step 1 — Day

Question:

> Which day would you like the room? mon | tue | wed | thu | fri | sat

Only `mon`, `tue`, `wed`, `thu`, `fri`, or `sat` are accepted.

An unrecognized answer does not advance the flow; the day question is re-asked.

### Step 2 — Time

Question:

> Which time? Morning, afternoon or evening.

Only `morning`, `afternoon`, or `evening` are accepted.

An unrecognized answer does not advance the flow.

### Step 3 — Number of people

Question:

> How many people? Enter a number from 1 to 8.

The party size is validated with a regular expression:

`\b(?:[1-8])\b`

This means the value must be a number from 1 to 8. Number words such as `four` are not accepted.

### Confirmation

After all three values are valid, the chatbot restates them:

> A room on {day}, {time}, for {n} people. Is that correct?

The booking is only treated as confirmed after `yes` or `y`.

### Exit

`bye` works at every booking step. It cancels the booking and clears:

- day
- time
- number of people
- current booking step

This prevents a later conversation from inheriting an old booking.

## Part C — Testing

The notebook runs the test set and prints the result produced by the code. The first eighteen cases are the required sample phrasings. Additional cases check capitalization, robustness, an unknown time/number, a misspelling, and the three supplied out-of-scope requests.

The notebook does **not** change the reported result to force a target score. It reports the result actually produced by the implementation.

## Failure diagnosis

The assignment identifies three Week 4 failure causes:

### 1. Keyword too short

A very short keyword is unsafe by itself because it may not contain enough information to identify an intent.

### 2. Missing word boundary

Without `\b`, a keyword can match inside another word. Word boundaries are therefore included in the intent patterns.

### 3. Genuinely shared word

Some words genuinely belong to more than one intent. The main example is `book`: it can relate to borrowing a book or booking a study room. The solution is to put the more specific `rooms` rule before `borrow`.

A misspelling is handled conservatively: if the supplied pattern cannot recognize it, the chatbot uses the fallback instead of guessing the user's intent.

## Known limitations

This is a deliberately simple rule-based chatbot.

- It recognizes only the phrases covered by the regular expressions.
- It does not understand arbitrary natural-language paraphrases.
- It does not correct spelling automatically.
- It does not connect to a real library database.
- It does not check real room availability.
- It does not store real reservations.
- The booking confirmation is an in-memory demonstration only.
- The number of people must be entered as a digit from 1 to 8.
- Out-of-scope requests are sent to the fallback rather than answered.

## Files

The Week 4 directory contains:

- `IHIMBAZWE_DIVINSE_Assignment_4.ipynb` — completed notebook
- `README.md` — project documentation
