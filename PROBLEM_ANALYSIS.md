# PROBLEM ANALYSIS: Mailman + Infomaniak SMTP Relay

## Date: 2024-12-30

## THE CORE PROBLEM

**Symptom**: All messages sent through Mailman mailing lists fail with:
```
550 5.7.1 Sender mismatch (in reply to end of DATA command)
```

**Key Facts**:
- Direct email FROM mailman@iaqi.org works perfectly (tested at 09:43:41, status=sent)
- All list messages fail with "Sender mismatch" 
- Infomaniak SMTP relay (mail.infomaniak.com:587) validates sender authenticity
- Envelope sender is successfully rewritten to mailman@iaqi.org (visible in logs: from=<mailman@iaqi.org>)
- Messages still rejected DESPITE correct envelope

## ROOT CAUSE HYPOTHESIS

Infomaniak validates **both** envelope sender AND message headers (From/Sender) for consistency.

Evidence:
1. Direct test `echo "test" | sendmail -f mailman@iaqi.org tech@iaqi.org` → SUCCESS
2. List messages with envelope from=<mailman@iaqi.org> → FAIL
3. smtp_header_checks patterns NOT triggering (no "replace:" in logs after 09:22:33)
4. Mailman likely preserves original sender's email in From header (e.g., huebli@gmail.com)
5. Infomaniak sees: Envelope=mailman@iaqi.org, From header=huebli@gmail.com → MISMATCH

## CONFIGURATION STATE

### What IS Working:
- Postfix SASL authentication to Infomaniak
- Email receiving via Fetchmail
- Mailman core functionality (accepts messages, processes them)
- Envelope sender rewriting via smtp_generic_maps
- HTTPS with Let's Encrypt
- Web interface at https://list.iaqi.org

### Current Configuration Files:

#### /etc/postfix/main.cf (relevant sections):
```
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_auth_enable = yes
smtp_sasl_tls_security_options = noanonymous
smtp_tls_security_level=encrypt

# Mailman LMTP transport
owner_request_special = no
transport_maps = hash:/var/lib/mailman3/data/postfix_lmtp
local_recipient_maps = proxy:unix:passwd.byname $alias_maps hash:/var/lib/mailman3/data/postfix_lmtp
relay_domains = ${{$compatibility_level} < {2} ? {$mydestination} : {}} hash:/var/lib/mailman3/data/postfix_domains

# Sender rewriting (REVERTED TO SIMPLEST)
smtp_generic_maps = regexp:/etc/postfix/smtp_generic
```

#### /etc/postfix/smtp_generic:
```
/@list\.iaqi\.org$/    mailman@iaqi.org
```

#### /etc/mailman3/mailman.cfg (relevant sections):
```
site_owner: tech@iaqi.org

[list.defaults]
delivery_status: enabled
```

## ATTEMPTED SOLUTIONS (in chronological order)

### 1. smtp_generic_maps (PARTIAL SUCCESS - CURRENT STATE)
**What**: Rewrite envelope sender from list addresses to mailman@iaqi.org
**Result**: Envelope correctly rewritten (confirmed in logs: from=<mailman@iaqi.org>)
**Why it failed**: Infomaniak also validates message headers, not just envelope

### 2. sender_canonical_maps (REDUNDANT)
**What**: Additional envelope rewriting mechanism
**Result**: Same as smtp_generic_maps (envelope level only)
**Why it failed**: Doesn't affect message headers

### 3. Disable VERP (NO EFFECT)
**What**: Disabled Variable Envelope Return Path to simplify envelope addressing
**Configuration**: 
```
verp_probes: no
verp_personalized_deliveries: no
verp_delivery_interval: 0
```
**Result**: No change in behavior
**Why it failed**: VERP was not the issue; envelope sender was already correct

### 4. header_checks (WRONG SCOPE)
**What**: Attempted to rewrite headers
**Result**: Doesn't work for outgoing SMTP (only works on incoming/cleanup daemon)
**Why it failed**: Wrong Postfix processing stage

### 5. smtp_header_checks (NOT TRIGGERING)
**What**: Rewrite From/Sender headers at SMTP stage
**Configuration**:
```
/^From:.*list\.iaqi\.org/i    REPLACE From: Mailman List <mailman@iaqi.org>
/^Sender:.*list\.iaqi\.org/i    REPLACE Sender: mailman@iaqi.org
```
**Result**: No "replace:" entries in mail.log (patterns not matching)
**Why it failed**: Mailman's From headers don't contain "@list.iaqi.org" - they contain the original sender's address (e.g., huebli@gmail.com)
**Pattern testing**: `postmap -q` confirmed patterns ARE valid and WOULD match if headers contained list addresses

### 6. from_is_list: munge (UNVERIFIED THEORY)
**What**: Configure Mailman to rewrite From headers to list addresses
**Expected behavior**: From header becomes "Original Sender via ListName <listname@list.iaqi.org>"
**Result**: PENDING - Not currently applied (reverted to simplest config)
**Theory**: If Mailman rewrites From to contain @list.iaqi.org, then smtp_header_checks WILL match and rewrite to mailman@iaqi.org

## WHY EACH ATTEMPT FAILED

The fundamental issue: **Message header rewriting requires the headers to contain a pattern we can match**

- smtp_generic_maps/sender_canonical_maps: Only rewrite envelope (MAIL FROM command)
- smtp_header_checks: Only matches headers that contain the pattern
- Mailman default behavior: Preserves original sender in From header
- Result: No pattern match → No header rewrite → Infomaniak sees mismatch

## THE THEORY FOR NEXT ATTEMPT

Make Mailman change the From header BEFORE Postfix sees it:
1. Mailman receives message from huebli@gmail.com to quantum@list.iaqi.org
2. Mailman rewrites From header to include list address (via from_is_list: munge)
3. Postfix smtp_header_checks sees "From: ... quantum@list.iaqi.org"
4. Pattern matches → Replace with "From: Mailman List <mailman@iaqi.org>"
5. Envelope already shows mailman@iaqi.org (smtp_generic_maps)
6. Infomaniak sees consistent sender → Accepts message

## NEXT DEBUGGING STEPS

1. **CONFIRM THE ROOT CAUSE** - Check actual message headers in Postfix queue:
   ```bash
   # Send test message to quantum@list.iaqi.org
   # Then on server:
   sudo postqueue -p  # get message ID
   sudo postcat -qh <MESSAGE_ID>
   # Look at the From: header - does it show original sender or list address?
   ```

2. If From header shows original sender (e.g., huebli@gmail.com):
   - Apply from_is_list: munge configuration
   - Send new test message
   - Check if From header now contains list address
   
3. If from_is_list works:
   - Add smtp_header_checks to rewrite to mailman@iaqi.org
   - Test delivery

4. If from_is_list doesn't work or isn't suitable:
   - Try alternative approaches (see below)

## ALTERNATIVE APPROACHES NOT YET TRIED

### Option A: Mailman anonymous_list setting
Configure Mailman to use a site-wide From address for ALL messages
```
[list.defaults]
anonymous_list: yes
```
- May lose original sender information
- Could affect reply-to functionality

### Option B: Run own mail server
Set up full MTA to accept outbound mail without validation
- User explicitly did NOT want this ("wait are we setting up our own mailserver now?")
- More complex, more maintenance

### Option C: Different SMTP relay provider
Find relay that doesn't validate sender headers as strictly
- Requires new email service
- May have other restrictions

### Option D: Postfix milter/policy server
Write custom sender rewriting at milter level
- Very complex
- Maintenance burden

### Option E: Contact Infomaniak Support
Ask exactly what validation they perform
- May reveal configuration options
- Could provide recommended Mailman settings

## SIMPLIFIED TEST CASE

To isolate the problem:

```bash
# This works (proven at 09:43:41):
echo "Test message" | sendmail -f mailman@iaqi.org tech@iaqi.org

# This fails (all list messages):
# - Send to quantum@list.iaqi.org
# - Mailman processes it
# - Envelope shows from=<mailman@iaqi.org> (CORRECT)
# - Headers show From: huebli@gmail.com (THEORY - causes rejection)
# - Infomaniak rejects due to mismatch
```

## LOGS EVIDENCE

### Last successful direct send (09:43:41):
```
Dec 30 09:43:41 ov-61454f postfix/smtp[43864]: 2FB574280F: to=<tech@iaqi.org>, 
relay=mail.infomaniak.com[83.166.143.45]:587, delay=0.34, delays=0.01/0/0.17/0.17, 
dsn=2.0.0, status=sent (250 2.0.0 Ok: queued as 4dgSp46tQ1zGtG)
```

### List message failures (multiple):
```
Dec 30 09:51:35 ov-61454f postfix/smtp[45808]: 59FCB406AD: to=<tech@iaqi.org>, 
relay=mail.infomaniak.com[2001:1600:0:aaaa::1:1]:587, delay=0.28, 
delays=0.01/0.02/0.1/0.15, dsn=5.7.1, status=bounced 
(host mail.infomaniak.com[2001:1600:0:aaaa::1:1] said: 550 5.7.1 Sender mismatch 
(in reply to end of DATA command))
```

### No header replacement activity:
```
$ grep "replace:" /var/log/mail.log | tail -20
# Last entries at 09:22:33 (before list messages)
# No "replace:" for any list messages
```

## STATUS: REVERTED TO SIMPLEST CONFIGURATION

The configuration has been simplified to just envelope rewriting (smtp_generic_maps).

All complex attempts (sender_canonical_maps, smtp_header_checks, from_is_list, VERP disabling) have been removed.

**Next step for smarter AI**: 
1. Examine actual message headers in Postfix queue to confirm root cause
2. Based on findings, apply targeted solution
3. Consider contacting Infomaniak support for guidance

---

**For the next AI**: 

The envelope rewriting works (verified in logs). The problem is likely header validation.

**START HERE**: Send a test message and examine the headers in the queue with `postcat -qh`. This will definitively show what's in the From header and confirm the mismatch theory.

Then apply the appropriate fix based on what you find.
