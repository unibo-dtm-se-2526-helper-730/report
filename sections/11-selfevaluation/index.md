---
title: Self-evaluation
has_children: false
nav_order: 12
---

# Self-evaluation

## Ilaria Colavita

### My role

This is an individual project, so I covered every role: I analysed the tax rules and the
broker formats, designed the architecture and the domain model, implemented backend and
frontend, wrote the tests, set up the CI/CD pipeline and the releases, and wrote this
report.

As stated in the disclaimer on the home page, I used an AI assistant mainly to generate
tests, to draft the data model and to write documentation. I reviewed what it produced
against the code and the tax rules, and in some cases this review found mistakes, both in
the code and in the documentation, that I then fixed.

### Strengths

- **The tax core is isolated and well tested.** The rules live in a framework-free core,
  checked by an architecture test, with 98% coverage of the engine and the tax loss
  carryforward and IVAFE calculators fully covered. The results on the sample files were
  checked against hand-computed values.
- **Every number can be explained.** Each tax amount can be traced back to the sales,
  the purchase lots and the transactions that produced it, which is what a user needs to
  trust the result and justify it.
- **The application handles the cases that are easy to get wrong**: LIFO for shares and
  weighted average for funds, government bonds at 12.5% (also inside ETFs), capital
  income never offset by losses, losses expiring after four years.
- **Releases are fully automated.** From a merged pull request to published images, tag
  and release notes, nothing is done by hand, and versions follow the commit history.
- **Privacy by design.** Everything runs on the user's computer and no data leaves it.

### Weaknesses

- **Limited scope.** Only two brokers and only euro transactions are supported; corporate
  actions (splits, mergers) and transactions before 2015 are not handled. Trade Republic
  has offered the administered regime in Italy since 2025, so its support is useful only
  for the accounts that remain under the self-reporting regime.
- **Some tax cases are simplified.** The interest accrued in the price of a bond is not
  separated from the gain, losses on fund units are valued without fees, and the IVAFE
  ignores the cash left on the trading account. These choices are reasonable for the
  typical user, but a tax advisor would treat them more precisely.
- **The declaration must be checked.** The rows of the Modello 730 follow my reading of the
  2026 instructions; some details, such as the columns of rigo T102, should be checked
  against the official forms every year.
- **The frontend is less tested than the backend**: its coverage is about 35%, and there
  are no automated end-to-end tests through the browser.
- **No access control.** The application has no login, which is fine on a personal
  computer, but it should not be exposed on a shared network.

### What I would do differently

I would write the tests of each tax rule from the official examples before the code, to
catch the misreadings of the rules earlier. I would also validate the declaration with a
tax advisor on a real case.