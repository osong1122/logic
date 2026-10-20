ATR SUBSCRIPTION MATCH - LOGIC AND DECISIONS
============================================
For every subscription coming up for renewal, the process works out the
dollars available to renew (ATR), finds the closed won deal that renewed it,
and measures how much of that ATR the deal filled.
It's needed because Salesforce doesn't always tie a renewal back to the
subscription it replaces. Deals get consolidated, migrated (e.g. self-hosted
to SaaS), converted, or re-papered under a different contract or partner.
-----------
ATR Build.sql runs top to bottom (about 1 minute) and rebuilds three tables
in analytics_sandbox.main:
  Step  Table             One row per                     What it holds
  ----  ----------------  ------------------------------  ----------------------------------
  1     atr_sub_base_t    subscription (all years)        ATR value, renewal value, in scope
  2     atr_renewals_t    opp line on a closed won deal   renewal dollars, local and USD
  3     atr_matched_t     subscription in scope           deal that renewed it, amount
                                                          filled, fill status
SCOPE
-----
  - Selling Item subscriptions ending 11/1/2025 or later, with no end-date cap.
  - The deal that created the subscription must be Closed Won or
    Consumption - Closed. Subscriptions off lost or consolidated deals aren't
    real ATR.
  - Replaced and terminated subscriptions stay in. We care whether each
    subscription was filled, not about point-in-time ATR.
  - Subscriptions from every year compete for renewal dollars, so a deal that
    renewed an older subscription isn't credited to a newer one.
ATR (WHAT EACH SUBSCRIPTION IS UP FOR)
--------------------------------------
  - Source: the opp line on the quote's own opportunity, annualized:
    total price x 12 / months on the opp line dates. We don't use ACV, because
    ACV understates multi-year deals.
  - Replaced subscriptions: valued at their original quantity. CPQ zeroes the
    quantity when a subscription is replaced.
  - One opp line, several subscriptions: the line is split by
    price x quantity share and flagged is_one_to_many.
  - No opp line: falls back to the subscription's own price x quantity.
  - Zero-priced subscriptions: $0 ATR.
  - Ramps: only the last segment of a product on a quote is up for renewal.
    Earlier segments are $0.
RENEWAL DOLLARS 
---------------------------------------
  - Source: opp lines on closed won deals, not the subscriptions they create.
    A deal counts the day it closes, even before contracting.
  - Value per line: the higher of the opp line ACV or its annualized total
    price. It's applied once per deal x product, so ramps don't double count;
    any uplift goes on the first segment.
  - Upgrade lines are excluded.
  - Non-CPQ lines (no quote line, mostly de-bookings) are kept and net
    against the deal's dollars. They're flagged is_cpq = false. De-booking
    logic will be handled separately.
  - Currency: everything is matched in USD at the deal's close-date rate,
    then converted back to the subscription's currency.
HOW A SUBSCRIPTION GETS TIED TO A DEAL
-------------------------------------------------------
  1. Direct
       A renewal quote line on the deal points at the subscription.
  2. Replacement
       A Replaced record on a closed won quote. Partial replacements only
       fill the replaced share.
  3. Conversion
       Maintenance subscription -> subscribed asset -> account asset
       converted on a closed won quote. It fills from the asset's product
       group, capped at the share that converted.
  4. Contract
       The subscription's contract renewal opp is closed won.
  5. Linked expansion
       The renewal was consolidated into another deal (followed up to 5 hops
       to a closed won deal), or closed lost and points at the deal that
       replaced it (related product opportunity).
  6. Account expansion - on time / late
       Another closed won deal on the same end customer. The end customer is
       the named end tenant on the deal, or the account if there isn't one.
       The deal has to:
         - close after the deal that created the subscription
         - run more than a month past the subscription's end
         - start no later than 180 days after it
       On time = starts within 90 days of the end.
       Late    = starts 90-180 days after.
  7. Account expansion - early
       Same as 6, but the deal starts more than 90 days before the
       subscription ends.
  Subscriptions whose own renewal is still open skip links 5-7, so they
  don't get pre-filled by unrelated deals.
FILLING RULES
-------------
  - One deal per subscription: its strongest link, then the deal with the
    most dollars in its product group.
  - Within a deal and product group, first come, first served: subscriptions
    line up by link priority, then end date, and each takes what's left until
    the deal runs out. There's no pro-rata split.
  - No fills across product groups: PAM, Identity Access, Secrets & AI, EPM,
    MIM, Maintenance and PS are separate.
  - Direct fills are capped at ATR. Anything above ATR, and whatever is left
    on a deal, is expansion.
TRADE-OFFS WE ACCEPTED FOR SIMPLICITY
-------------------------------------
  - A subscription only takes one deal. If that deal runs short, the
    subscription stays partial and doesn't look for a second deal.
  - Matching is at product group level, with no product-family-first step.
  - A partially replaced subscription uses one link only: the replacement for
    the replaced share, or the direct renewal for the share it kept.
CURRENT RESULTS (SUBSCRIPTIONS ENDING 11/1/25 - 9/30/26, USD)
-------------------------------------------------------------
                                                           Subs     ATR $
                                                           -----    -----
  Natural system link
    (quote line, replacement, conversion, contract)         ~81%     ~83%
  + consolidated into / related product opp                 ~85%     ~87%
  Fuzzy end-customer match                                   ~1%      ~1%
  No connection (churn, open renewals, missed links)        ~14%     ~11%
  ATR:     $734M
  Filled:  $590M (80%)
  Checks:  no subscription overfilled, one deal per subscription
OPEN ITEMS
----------
  - Ops sign-off on atr_matched_t before loading into SFDC.
  - De-booking logic (to be handled separately).
  - About 40 subscriptions off lost or consolidated deals can still compete
    for renewal dollars. It's a one-line fix.
  - Review the long tail of unconnected subscriptions. Candidates for new
    rules: parent-account rollup, MSP mapping, product successor mapping.
