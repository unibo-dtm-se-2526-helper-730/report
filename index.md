---
title: Home
layout: home
has_children: false
nav_order: 1
---

# Helper-730

### Authors

- [Ilaria Colavita](mailto:ilaria.colavita@studio.unibo.it)

## Abstract

Helper Dichiarazione 730 is a web application that helps Italian tax residents who invest through foreign brokers fill in the investment sections of their annual tax return (Modello 730). Under the self-reporting regime (_regime dichiarativo_), these investors must work out on their own the substitute tax on capital gains and investment income, the tax loss carryforward (_zainetto fiscale_) and the wealth tax on foreign financial assets (IVAFE). Doing it by hand from a broker statement is slow and error-prone.

The user uploads the CSV exported by the broker (Revolut or Trade Republic). The application recognises the format, imports the transactions and asks the user to classify each instrument once: shares, bonds, government bonds, UCITS ETFs and so on. From then on, every tax year is computed from the whole transaction history. Sales are matched to purchases with the LIFO method required by Italian law (weighted average cost for fund
units), gains and income are split into miscellaneous and capital income, losses are offset and carried forward for four years, and IVAFE is computed pro rata on the days each holding was owned. Every figure can be traced back to the transactions that produced it.

The output is a summary of the values to enter in quadri T, M and W of the Modello 730.
The interface is available in Italian and English.

The system is made of a React single-page application, a Spring Boot REST API and a PostgreSQL database. The tax engine is a framework-free core, covered by unit tests and by tests on sample broker exports, and new brokers can be added without changing it.

## Disclaimer

During the preparation of this work, the author used Claude (Anthropic) mainly to generate the automated tests, to draft the data model of both the backend and the frontend, and to write the documentation.

After using this tool, the author reviewed and edited the content as needed and takes full responsibility for the content of the final report/artifact.
