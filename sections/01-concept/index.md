---
title: Concept
has_children: false
nav_order: 2
---

# Concept

## Type of product

Helper Dichiarazione 730 is a **web application with a graphical user interface**. It is
made of a single-page application that runs in the browser and a REST web service that
holds the data and does the tax calculations. The web service has a versioned, documented
API (OpenAPI), so it could also be used by other clients or scripts, but the browser
interface is the intended way to use it.

The application is meant to be run by the user on their own machine (or a private server)
with Docker Compose, rather than offered as a public multi-user service. For this reason it
has no user accounts: one installation serves one taxpayer.

## Users

The target users are **Italian tax residents who invest through foreign brokers** that do
not act as tax withholding agents in Italy, such as Revolut. These investors fall under the
self-reporting regime and must compute their investment taxes themselves. They are not tax
experts: they know what they bought and sold, but not how the law turns it into the boxes
of the tax return.

## When and how they use it

Use is seasonal. Most interaction happens **once a year**, when the tax return is prepared. A typical session is:

1. download the transaction CSV from the broker's app or website;
2. upload it to the application;
3. classify the instruments that appear for the first time;
4. enter the prices at 31 December needed for IVAFE, and any losses from before the first
   import;
5. review the results of the year and copy the values into the Modello 730.

Some users may also import their transactions during the year, to see the tax due so far
or the current portfolio. Since imports are idempotent, the same file can be uploaded again
without creating duplicates.

Interaction happens in a **desktop or laptop browser**. The tasks involve files exported
from the broker and long tables of figures, which fit a large screen better than a phone.
The interface is available in Italian and English.

## Use cases

There is a single role, the **taxpayer**. The main use cases
are:

| Use case                       | Description                                                                                                                                                                          |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Import transactions            | Upload a Revolut or Trade Republic CSV; the format is recognised, each row is validated and a report lists imported rows, duplicates and rows with errors.                           |
| Classify instruments           | Assign each instrument its asset class (share, bond, government bond, UCITS ETF, …), plus the share of government bonds for bond ETFs and the foreign withholding rate on dividends. |
| Browse transactions            | Look at the imported operations, filtered by year.                                                                                                                                   |
| Enter losses of previous years | Record losses realized before the first import, which open the tax loss carryforward.                                                                                                |
| Enter year-end prices          | Record the price of each holding at 31 December, used to value it for IVAFE.                                                                                                         |
| Review the tax of a year       | See the substitute tax by income category, the tax loss carryforward and, for every sale, the purchase lots it consumed.                                                             |
| Review IVAFE                   | See the wealth tax on foreign assets, holding period by holding period.                                                                                                              |
| Get the declaration summary    | See the values to write in quadri T, M and W of the Modello 730, with warnings for the cases the application cannot handle.                                                          |
| View the portfolio             | See the current holdings at cost and their split by asset class.                                                                                                                     |
| Delete the data                | Remove all imported transactions, and optionally the instruments, to start again.                                                                                                    |

## Stored data

The application has to keep the user's data between sessions, because the result of a
year depends on all the years before it: purchase lots carried over from earlier years and
losses still usable. The data is stored in a **PostgreSQL database** on the user's
machine:

- the imported **transactions**, with the import they come from (file, broker, date);
- the **instruments**, with the classification entered by the user;
- the **prices at 31 December** entered for each year;
- the **losses of previous years** entered by hand.

Tax results are not stored: they are computed again from these data at each request, so
they always reflect the latest imports and classifications.
