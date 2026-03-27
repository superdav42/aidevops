---
description: Email deliverability testing - spam analysis, provider checks, inbox placement
mode: subagent
tools:
  read: true
  write: false
  edit: false
  bash: true
  glob: true
  grep: true
  webfetch: true
---

# Email Delivery Testing

<!-- AI-CONTEXT-START -->

## Quick Reference

- **Purpose**: Spam content analysis, provider-specific deliverability, inbox placement testing
- **Script**: `email-delivery-test-helper.sh [command] [options]`
- **Tools**: dig, openssl, curl, nc (required); swaks, spamassassin (optional)
- **Related**: `email-health-check-helper.sh` (DNS auth), `email-test-suite-helper.sh` (design rendering)

```bash
email-delivery-test-helper.sh spam-check newsletter.html       # Spam content analysis
email-delivery-test-helper.sh spamassassin newsletter.html      # SpamAssassin (if installed)
email-delivery-test-helper.sh providers example.com             # All provider checks
email-delivery-test-helper.sh gmail example.com                 # Gmail-specific
email-delivery-test-helper.sh outlook example.com               # Outlook-specific
email-delivery-test-helper.sh yahoo example.com                 # Yahoo-specific
email-delivery-test-helper.sh seed-test example.com             # Seed-list testing guide
email-delivery-test-helper.sh send-test me@example.com test@gmail.com smtp.example.com 587
email-delivery-test-helper.sh warmup example.com                # Warm-up schedule
email-delivery-test-helper.sh report example.com                # Full deliverability report
```

<!-- AI-CONTEXT-END -->

## Spam Content Analysis

Analyses email HTML/text for content-level spam signals. Produces a score (0-100):

| Score | Rating | Meaning |
|-------|--------|---------|
| 0-10 | CLEAN | Unlikely to trigger spam filters |
| 11-25 | LOW RISK | Minor issues, should pass most filters |
| 26-50 | MEDIUM RISK | May trigger filters in some providers |
| 51-75 | HIGH RISK | Likely to be flagged as spam |
| 76-100 | CRITICAL | Will almost certainly be flagged |

**What it checks:**

- **Subject line** — ALL CAPS, excessive punctuation, financial/prize language
- **High-risk phrases** — "act now", "buy now", "click here", "free gift", etc.
- **Medium-risk phrases** — "bargain", "discount", "exclusive deal", etc.
- **Structural signals** — Image-to-text ratio, URL count, shortened URLs, hidden text, JavaScript, form elements
- **Compliance** — Unsubscribe link, physical address (CAN-SPAM)

## Provider-Specific Deliverability

| Provider | Score | Key checks |
|----------|-------|------------|
| **Gmail** | /8 | SPF, DKIM, DMARC, one-click unsubscribe (Feb 2024), PTR, Google Postmaster Tools, ARC headers |
| **Outlook** | /7 | SPF, DKIM, DMARC, MTA-STS, blacklist status, Microsoft SNDS |
| **Yahoo/AOL** | /5 | SPF, DKIM, DMARC, one-click unsubscribe (Feb 2024) |

### Feb 2024 Bulk Sender Requirements (>5000 emails/day)

| Requirement | Gmail | Yahoo |
|-------------|-------|-------|
| SPF | Required | Required |
| DKIM | Required | Required |
| DMARC (p=quarantine+) | Required | Required |
| One-click unsubscribe | Required | Required |
| Spam rate < 0.3% | Required | Required |
| PTR records | Required | Recommended |

## Inbox Placement & Warm-Up

Seed-list testing, SMTP send tests (via swaks or openssl), and warm-up scheduling for new IPs/domains.

### Warm-Up Schedule

| Day | Daily Volume | Notes |
|-----|-------------|-------|
| 1-2 | 50 | Most engaged contacts only |
| 3-4 | 100 | Monitor bounce/complaint rates |
| 5-6 | 250 | Check Google Postmaster Tools |
| 7-8 | 500 | Review inbox placement |
| 9-10 | 1,000 | Expand to broader audience |
| 11-14 | 2,500 | Continue monitoring |
| 15-21 | 5,000 | Steady increase |
| 22-28 | 10,000 | Approaching normal volume |
| 29+ | 25,000+ | Full volume (if metrics healthy) |

## Integration with Other Email Tools

| Tool | Focus |
|------|-------|
| `email-health-check-helper.sh` | DNS authentication (SPF, DKIM, DMARC) with graded scoring |
| `email-test-suite-helper.sh` | Design rendering + delivery infrastructure |
| `email-delivery-test-helper.sh` | Spam content + provider deliverability + inbox placement |

**Recommended workflow:** DNS auth check → spam content analysis → provider deliverability → design rendering → send test emails.

## External Services

| Service | Purpose | URL |
|---------|---------|-----|
| **Google Postmaster** | Gmail reputation monitoring | postmaster.google.com |
| **Microsoft SNDS** | Outlook reputation monitoring | sendersupport.olc.protection.outlook.com |
| **mail-tester.com** | Deliverability scoring (free) | mail-tester.com |
| **GlockApps** | Inbox placement testing | glockapps.com |
| **Mailtrap** | Email sandbox | mailtrap.io |
| **Mailreach** | Warm-up automation | mailreach.co |
| **InboxAlly** | Warm-up automation | inboxally.com |

## Related

- `services/email/email-health-check.md` - DNS authentication checks
- `services/email/email-testing.md` - Design rendering and delivery testing
- `services/email/ses.md` - Amazon SES integration
- `content/distribution/email.md` - Email content strategy
