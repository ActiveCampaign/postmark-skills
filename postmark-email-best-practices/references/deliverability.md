# Email Deliverability

## Authentication Records

Every sending domain must have all three authentication records configured. Missing or broken records are the most common cause of email landing in spam.

### SPF (Sender Policy Framework)

SPF authorizes specific mail servers to send on behalf of your domain. Add Postmark to your SPF TXT record:

```
v=spf1 include:spf.mtasv.net ~all
```

If you already have an SPF record, add Postmark's include to it — do not create a second record:

```
v=spf1 include:existing-provider.com include:spf.mtasv.net ~all
```

**Rules:**
- Only one SPF TXT record per domain — merge all senders into one record
- `~all` (soft fail) is recommended over `-all` (hard fail), which can block legitimate forwarded email
- SPF has a 10 DNS lookup limit — flatten includes if you're near the limit

### DKIM (DomainKeys Identified Mail)

DKIM adds a cryptographic signature to every email, proving it hasn't been tampered with. When you verify a sending domain in Postmark, a DKIM CNAME is provided — add it to your DNS:

```
pm._domainkey.yourdomain.com  CNAME  pm.mtasv.net
```

Postmark rotates DKIM keys automatically. No manual key rotation needed.

### DMARC (Domain-based Message Authentication, Reporting, and Conformance)

DMARC tells receiving servers what to do when SPF or DKIM fails, and enables reporting back to you.

**Start with monitoring (safe, no impact on delivery):**

```
_dmarc.yourdomain.com  TXT  "v=DMARC1; p=none; rua=mailto:dmarc@yourdomain.com"
```

**After confirming legitimate traffic passes, move to quarantine:**

```
_dmarc.yourdomain.com  TXT  "v=DMARC1; p=quarantine; pct=100; rua=mailto:dmarc@yourdomain.com"
```

**Then enforce:**

```
_dmarc.yourdomain.com  TXT  "v=DMARC1; p=reject; pct=100; rua=mailto:dmarc@yourdomain.com"
```

| Parameter | Description |
|-----------|-------------|
| `p` | Policy: `none` (monitor only), `quarantine` (send to spam), `reject` (block) |
| `pct` | Percentage of failing mail the policy applies to (0–100) |
| `rua` | Address for aggregate reports (daily summaries) |
| `ruf` | Address for forensic reports (individual failures) |

**Verification tools:**
- [MXToolbox SuperTool](https://mxtoolbox.com/SuperTool.aspx) — look up SPF, DKIM, DMARC
- [Mail Tester](https://www.mail-tester.com) — send a test email for a deliverability score
- [Postmark's DMARC Digests](https://dmarc.postmarkapp.com) — free DMARC reporting tool

---

## Sender Reputation

Authentication proves identity. Reputation determines whether your email is trusted.

### Reputation Factors

| Factor | Impact | Action |
|--------|--------|--------|
| Bounce rate | High negative | Remove hard bounces immediately |
| Spam complaint rate | Very high negative | Suppress complainers immediately |
| Engagement (opens, clicks) | Positive | Send to engaged recipients; segment out inactive |
| Sending consistency | Moderate positive | Consistent volume beats unpredictable spikes |
| Domain age | Moderate positive | New domains need warm-up |
| Content quality | Positive | Relevant, expected email is rarely marked as spam |

### Thresholds to Stay Below

| Metric | Warning | Critical — Stop Sending |
|--------|---------|------------------------|
| Bounce rate | > 2% | > 4% |
| Spam complaint rate | > 0.04% | > 0.08% |

---

## Domain Warm-up

New sending domains have no reputation. Sending large volumes immediately triggers spam filters. Gradually build volume to establish trust with inbox providers.

### Postmark Recommended Warm-up Schedule

| Day | Max per Day | Max per Hour |
|-----|-------------|--------------|
| 1 | 150 | — |
| 2 | 250 | — |
| 3 | 400 | — |
| 4 | 700 | 50 |
| 5 | 1,000 | 75 |
| 6 | 1,500 | 100 |
| 7 | 2,000 | 150 |

After day 7, increase volume gradually — no more than 2x per week.

**Warm-up best practices:**
- Start with your most engaged recipients (recent signups, active users)
- Spread sends evenly throughout the day — avoid hourly spikes
- Monitor bounce and complaint rates daily
- Pause and investigate if bounce rate exceeds 4% or complaint rate exceeds 0.08%

### Signs the Warm-up Is Working

- Bounce rate stays below 2%
- Complaint rate stays below 0.04%
- Open rates match baseline expectations for your audience
- Emails landing in inbox, not spam folder

---

## Diagnosing Deliverability Issues

### Email Landing in Spam

1. Verify SPF, DKIM, and DMARC are all passing (use MXToolbox)
2. Check bounce and complaint rates in the Postmark dashboard
3. Review list quality — old or purchased lists have high invalid/disengaged addresses
4. Check content — excessive images, spam trigger words, broken HTML
5. Check if your domain or IP is on a blocklist (MXToolbox Blacklist Check)

### Emails Not Arriving at All

1. Confirm the `From` address uses a verified domain or sender signature in Postmark
2. Check the Postmark activity log for bounce or block status
3. Review the SMTP response in the bounce dump for the recipient server's reason
4. Test with the recipient on a different domain to isolate whether it's domain-specific

### Low Open Rates

1. Check spam folder placement — send a test to an inbox monitoring service
2. Review subject line length (30–50 characters is optimal for mobile)
3. Confirm sender name is recognizable — recipients recognize names, not addresses
4. Note: Apple Mail Privacy Protection (MPP) artificially inflates open rates. Use click rate as the primary engagement signal.
