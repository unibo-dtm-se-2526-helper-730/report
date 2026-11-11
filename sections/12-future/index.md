---
title: Future work
has_children: false
nav_order: 13
---

# Known issues and future work

## What is missing

- **Only two brokers.** Revolut and Trade Republic are supported; exports of other brokers
  (Interactive Brokers, Degiro, Scalable Capital, …) are rejected.
- **Only euro.** Transactions in other currencies are rejected: converting them would need
  the official exchange rates of the Bank of Italy for each day.
- **No corporate actions.** Splits, mergers, spin-offs and stock dividends are reported as
  errors: the holdings involved must be fixed by hand.
- **No transactions before 2015.** Earlier years follow different rates and loss rules; if
  the file contains such transactions, no year can be computed.
- **Cash is ignored by IVAFE.** The balance left on the broker account is not computed;
  the IVAFE page only reminds the user to declare it when it is above 5,000 €.
- **Part of the declaration is left to the user.** Only the rows that the application can
  compute are shown (e.g. T11 and T102 in quadro T); the others, such as revalued costs or
  losses certified by Italian intermediaries, must be filled in by hand.
- **No login.** The application is meant for a single user on their own computer and must
  not be exposed on a shared network.

## What does not work as it should

- **Trade Republic accounts under the administered regime.** Since 2025 Trade Republic can
  withhold Italian taxes itself. If an export contains taxes withheld by the broker, the
  application subtracts them from the amounts of sales and dividends, so it computes
  lower gains and could count the same tax twice. Such accounts are outside the scope of
  the application, but it does not detect them.
- **Simplified tax cases.** The interest accrued in the price of a bond is treated as part
  of the gain instead of as capital income, and losses on fund units are valued without
  fees, so they are slightly smaller than they could be.
- **Rows of the Modello 730 to be confirmed.** The layout follows the 2026 instructions as I
  read them; some details, such as the columns of rigo T102, should be checked against the
  official form every year.
- **Released images are hard to run.** The images are published, but the repository has no
  Compose file that uses them: users have to build the images or adapt the configuration.
- **Accessibility.** In the audit trail of the summary, a sale can be expanded into its
  lots only with the mouse.
- **Frontend tests.** About a third of the frontend code is covered by tests, and there is
  no automated test through the browser.

## Future developments

- **More brokers and currencies**, with exchange rates downloaded from the Bank of Italy.
- **IVAFE on cash**, by rebuilding the daily balance of each broker account from its
  transactions (average balance, threshold of 5,000 €, fixed tax of 34.20 €).
- **Detection of the administered regime**, warning when an export contains taxes already
  withheld by the broker.
- **Bonds done properly**: separation of the accrued interest and of the issue discount.
- **Export of the declaration** as a PDF or a spreadsheet, to keep with the tax documents.
- **Modello Redditi PF**, for the users who cannot file the 730 (quadri RT, RM and RW).
- **End-to-end tests** of the user interface (for example with Playwright) and a
  Compose file to run the released images with one command.
- **Version 1.0.0**, once the supported features and the data format are considered stable.