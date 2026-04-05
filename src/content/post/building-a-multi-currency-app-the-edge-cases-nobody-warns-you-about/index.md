---
author: "Alfero Chingono"
title: "Building a Multi-Currency App: The Edge Cases Nobody Warns You About"
date: 2025-10-22T09:00:00Z
draft: true
description: "How Famorize handles global users: floating point errors, exchange rate drift, and the UX of money across borders."
slug: building-a-multi-currency-app-the-edge-cases-nobody-warns-you-about
tags: [
"Famorize",
"Multi-Currency",
"Financial Tech",
"Database Design",
"Build in Public",
"Software Engineering"
]
categories: [
"Craft",
"Build in Public"
]
image: "cover.png"
---

When I started building [Famorize](https://www.famorize.com), I thought multi-currency support would be a weekend feature. "Just store the currency code and multiply by the exchange rate, right?"

Wrong.

Dealing with money in software is a masterclass in edge cases, data integrity, and precision. If you're building an app that handles more than one currency, especially one designed for families across borders, the complexity stacks up fast.

Here are the four big lessons I learned from building Famorize's multi-currency engine.

## 1. Never, Ever Use Floating Point Numbers

This is the "Day 1" rule for financial software, yet I still see it in production. Floating point numbers (like `double` or `float` in most languages) are inherently imprecise for decimals. $0.1 + 0.2$ in IEEE 754 floating point is not $0.3$; it's $0.30000000000000004$.

In Famorize, everything is stored as **integers in the smallest unit** (e.g., cents for USD, yen for JPY, satoshis for BTC). If a transaction is $10.50, it is stored as `1050`. This eliminates rounding errors at the database and application levels.

## 2. Exchange Rate "Drift" and Historic Validity

Exchange rates are not static. If a user recorded a $50 CAD transaction today, it might be worth $37 USD. If they look at that same transaction next year, it should still be $50 CAD. But what was its USD value *at the time*?

To handle this, Famorize doesn't just store the transaction. It stores:

1.  **The Base Amount** (in the user's home currency).
2.  **The Original Amount** (in the transaction currency).
3.  **The Exchange Rate Used** (at the time of the transaction).

This "Snapshot Pattern" ensures that reports are accurate to the moment the money actually moved, not the current market rate.

## 3. The "Yen Problem" (Zero-Decimal Currencies)

Most developers assume a currency always has two decimal places. This is a trap.

While USD, CAD, and EUR have two (cents), currencies like JPY (Japanese Yen) have zero, and others like BHD (Bahraini Dinar) have three. If your UI or database assumes `integer / 100` is the decimal value, you will break for users in Japan or Bahrain.

Famorize uses a **Currency Metadata** table that defines the "Scale" (decimal places) for every supported currency. The display layer is responsible for formatting based on this scale.

## 4. The UX of "Which Currency?"

If a user in Toronto is sending money to a family member in Nairobi, which currency should the input field show?

The UX pattern that worked best for me was to always let the user select the **Transaction Currency** while showing a real-time "estimated conversion" into their **Home Currency**. That gives them a clearer sense of how much they are spending in the units they understand.

## The Result

Building a multi-currency app taught me that "precision" is not just a technical requirement; it is a trust requirement. If a financial app is off by even one cent, the user's trust is broken.

Famorize now handles dozens of currencies with an integer-based engine that I can trust with real family finances. It took more than a weekend, but the lessons were worth the effort.

---
*Related reading:*
- [OAuth2 and OIDC for Solo Developers: A Practical Setup With Docker](/blog/2025/09/15/oauth2-and-oidc-for-solo-developers-a-practical-setup-with-docker/)
- [Why I Started Building My Own DevOps Platform](/blog/2025/02/15/why-i-started-building-my-own-devops-platform-and-what-i-learned/)
