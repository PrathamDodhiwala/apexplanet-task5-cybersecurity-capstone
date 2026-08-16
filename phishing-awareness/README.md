# Phishing Awareness Simulation - Task 5 Phase 6

## Objective
Demonstrate, in a fully local and harmless environment, how a credential-phishing
page manipulates a user into entering sensitive information, and teach recognition
of the visual and psychological red flags that expose such attacks before any
credentials are ever typed.

## Threat Model
Real-world credential phishing typically impersonates a trusted service (bank,
email provider, internal company portal) and relies on:
- Urgency/fear to short-circuit careful thinking ("your account will be suspended")
- Visual trust cues copied from the real brand (logo, colors, layout)
- A convincing but wrong domain, often only spottable by checking the actual URL bar
- A generic, non-personalized greeting, since attackers rarely have the victim's real name
- A request to "verify" or "confirm" credentials via a link, rather than the user
  navigating to the real site directly

The captured credentials are then used for account takeover, further phishing, or
sold on to other attackers.

## Simulation Design
This page (index.html) intentionally reproduces the above red flags in a
controlled way:
- A persistent red banner clearly marking it as a training simulation
- Urgent, fear-based language ("suspended in 24 hours")
- A fake "SecureBank" brand with no real affiliation
- A visible red-flags panel directly below the fake login form, explaining each
  manipulation technique as the user sees it
- A "Verify Account" button that does not transmit any data anywhere - it only
  displays a message confirming the simulation and what would have happened with
  a real attacker's page. Confirmed via browser DevTools' Network tab showing no
  request fires on submit.

The page is served only on the local host-only lab network (192.168.56.10) and
was never exposed to any real user or the internet.

## Lessons Learned
- Urgency is a manipulation tactic, not a legitimate signal - real institutions
  don't threaten account suspension via unsolicited email links.
- Always check the actual URL/domain in the address bar rather than trusting the
  page's visual branding.
- Never enter credentials via a link from an email or message - navigate to the
  service directly instead.
- A generic greeting on a message claiming to be from your bank/provider is a
  strong indicator of a mass-targeted phishing attempt, not a personal, verified
  communication.

## Safety Notes
- No real credentials were ever collected, stored, or transmitted.
- No real-world targets, brands, or individuals were used.
- Simulation is confined to the isolated lab network and was not distributed.
