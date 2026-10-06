# Amazon SES Setup for StokSmart Campaign Email

## 1. Overview

This document describes the Amazon SES setup used for the campaign domain:

```text
stoksmartmail.com
```

The architecture is:

```text
Snov.io
  ↓
Amazon SES SMTP
  ↓
Outbound campaign email

Replies
  ↓
Microsoft 365 / Outlook
```

Amazon SES is used only for outbound sending. Microsoft 365 can be added later for inbound reply handling through IMAP.

---

## 2. AWS SES Region

Amazon SES was configured in:

```text
Region: Asia Pacific (Mumbai)
Region code: ap-south-1
```

The SES account is already in **Production**.

Current account limits:

```text
Daily sending quota: 50,000 emails/day
Maximum sending rate: 14 emails/second
```

This is sufficient for the current campaign requirement.

---

## 3. Add the Campaign Domain to Amazon SES

In AWS Console:

```text
Amazon SES
→ Configuration
→ Identities
→ Create identity
```

Choose:

```text
Identity type: Domain
Domain: stoksmartmail.com
```

For DKIM:

```text
DKIM: Enabled
DKIM method: Easy DKIM
DKIM key length: 2048-bit
```

Create the identity.

---

## 4. Configure DKIM in Namecheap

After creating the SES identity, AWS provides DKIM DNS records.

Normally SES Easy DKIM provides 3 CNAME records.

Example:

```text
abc123._domainkey.stoksmartmail.com
→ abc123.dkim.amazonses.com
```

In Namecheap, go to:

```text
Domain List
→ stoksmartmail.com
→ Advanced DNS
```

For each SES CNAME record, add:

```text
Type: CNAME
Host: abc123._domainkey
Value: abc123.dkim.amazonses.com
TTL: Automatic
```

Important:

Do not enter the full domain in the Namecheap Host field.

For example, if AWS shows:

```text
abc123._domainkey.stoksmartmail.com
```

enter only:

```text
abc123._domainkey
```

Namecheap automatically appends:

```text
.stoksmartmail.com
```

Repeat this for all SES DKIM records.

---

## 5. Verify the SES Domain

Return to:

```text
Amazon SES
→ Configuration
→ Identities
→ stoksmartmail.com
```

Wait until the identity status becomes:

```text
Verified
```

Also confirm DKIM status is:

```text
Successful
```

DNS verification may take several minutes.

---

## 6. Configure Custom MAIL FROM

After the domain is verified:

```text
Amazon SES
→ Configuration
→ Identities
→ stoksmartmail.com
→ MAIL FROM domain
→ Edit
```

Configure:

```text
MAIL FROM domain:
mail.stoksmartmail.com
```

Use a subdomain rather than the root domain.

AWS provides an MX record and an SPF TXT record for this subdomain.

For Mumbai, the MX destination normally uses:

```text
feedback-smtp.ap-south-1.amazonses.com
```

Add the exact values shown by AWS.

Example Namecheap records:

```text
Type: MX
Host: mail
Priority: 10
Value: feedback-smtp.ap-south-1.amazonses.com
```

and:

```text
Type: TXT
Host: mail
Value: v=spf1 include:amazonses.com ~all
```

Use the values shown in your SES console if they differ.

---

## 7. Add DMARC

Add a DMARC TXT record in Namecheap.

Initial configuration:

```text
Type: TXT
Host: _dmarc
Value: v=DMARC1; p=none;
```

This allows monitoring without rejecting messages.

Later, after confirming SPF and DKIM alignment, the policy can be strengthened to:

```text
p=quarantine
```

and eventually:

```text
p=reject
```

---

## 8. Create SES SMTP Credentials

In Amazon SES:

```text
Amazon SES
→ SMTP settings
→ Create SMTP credentials
```

Create a dedicated SMTP credential for Snov.io.

Example name:

```text
snov-production
```

AWS generates:

```text
SMTP username
SMTP password
```

Store both securely.

Important:

These are SES SMTP credentials, not the normal AWS Console login or IAM password.

---

## 9. SES SMTP Details

For the Mumbai region:

```text
SMTP host:
email-smtp.ap-south-1.amazonaws.com

Port:
587

Encryption:
STARTTLS / TLS

Username:
<SES SMTP username>

Password:
<SES SMTP password>
```

SES SMTP credentials are region-specific.

---

## 10. Configure Amazon SES in Snov.io

In Snov.io:

```text
Email Accounts
→ Add email account
→ SMTP / Amazon SES
```

Configure the sender.

Example:

```text
From name:
StokSmart Sales

From email:
sales1@stoksmartmail.com
```

SMTP settings:

```text
SMTP host:
email-smtp.ap-south-1.amazonaws.com

SMTP port:
587

Encryption:
TLS / STARTTLS

SMTP username:
<SES SMTP username>

SMTP password:
<SES SMTP password>
```

The SMTP username is not the sender email address.

The sender address can be:

```text
sales1@stoksmartmail.com
```

while authentication uses the SES SMTP username and password.

---

## 11. Multiple Sender Addresses

Because the whole domain is verified in SES, multiple sender addresses under the domain can use the same SES SMTP credentials.

Examples:

```text
sales1@stoksmartmail.com
sales2@stoksmartmail.com
sales3@stoksmartmail.com
sales4@stoksmartmail.com
sales5@stoksmartmail.com
```

Each sender can be added separately in Snov.io.

All can use:

```text
SMTP host:
email-smtp.ap-south-1.amazonaws.com

SMTP username:
<same SES SMTP username>

SMTP password:
<same SES SMTP password>
```

Only the visible From address changes.

---

## 12. IMAP / Reply Handling

For the initial setup, IMAP was left empty.

This does not affect outbound sending.

The current outbound flow is:

```text
Snov.io
  ↓
Amazon SES
  ↓
Recipient
```

Later, Microsoft 365 can be configured for inbound replies:

```text
Recipient reply
  ↓
Microsoft 365
  ↓
Shared campaign inbox
  ↓
Snov.io IMAP
```

Typical Microsoft 365 IMAP settings are:

```text
IMAP host:
outlook.office365.com

Port:
993

Encryption:
SSL/TLS
```

OAuth / Modern Authentication is preferred.

---

## 13. Test Outbound Sending

To test the setup in Snov.io:

1. Add your own external email address as a test prospect.
2. Create a one-step campaign.
3. Use the SES-connected sender.
4. Send:

```text
Subject: Hello World

Body:
Hello World
```

5. Start the campaign.
6. Confirm the message arrives.

---

## 14. Verify Email Authentication

After receiving the test message, inspect the message headers.

Confirm:

```text
SPF: PASS
DKIM: PASS
DMARC: PASS
```

Also confirm the visible From address is correct:

```text
sales1@stoksmartmail.com
```

---

## 15. Final Architecture

```text
stoksmartmail.com
│
├── Namecheap DNS
│   ├── SES DKIM CNAME records
│   ├── SES MAIL FROM MX record
│   ├── SES MAIL FROM SPF record
│   └── DMARC
│
├── Amazon SES
│   ├── Region: ap-south-1
│   ├── Domain identity: stoksmartmail.com
│   ├── Easy DKIM
│   ├── Custom MAIL FROM
│   └── SMTP credentials
│
└── Snov.io
    ├── sales1@stoksmartmail.com
    ├── sales2@stoksmartmail.com
    ├── sales3@stoksmartmail.com
    └── SES SMTP
```

---

## 16. Current SES Cost Estimate

At approximately:

```text
5,000 emails/day
```

Monthly volume is approximately:

```text
5,000 × 30
= 150,000 emails/month
```

At approximately:

```text
$0.10 per 1,000 outbound emails
```

the estimated SES sending cost is:

```text
150 × $0.10
= approximately $15/month
```

This excludes optional features such as dedicated IPs and additional data-transfer charges.

---

## 17. Go-Live Checklist

- [ ] `stoksmartmail.com` created as SES identity
- [ ] SES region is `ap-south-1`
- [ ] Domain status is Verified
- [ ] DKIM status is Successful
- [ ] All DKIM CNAME records exist in Namecheap
- [ ] Custom MAIL FROM configured
- [ ] MAIL FROM MX record added
- [ ] MAIL FROM SPF record added
- [ ] DMARC record added
- [ ] SES SMTP credentials created
- [ ] Snov.io SMTP account configured
- [ ] Test email successfully sent
- [ ] SPF passes
- [ ] DKIM passes
- [ ] DMARC passes
- [ ] Additional Snov.io sender accounts configured
- [ ] Microsoft 365 IMAP added later if reply tracking is required

---

## 18. Key Configuration Summary

```text
Domain:
stoksmartmail.com

AWS Region:
ap-south-1

SES SMTP Host:
email-smtp.ap-south-1.amazonaws.com

SMTP Port:
587

Security:
STARTTLS / TLS

SMTP Authentication:
SES SMTP username + SES SMTP password

MAIL FROM:
mail.stoksmartmail.com

Outbound:
Snov.io → Amazon SES

Inbound:
To be configured later through Microsoft 365
```
