---
layout: page
title: Rejecting RPKI Invalid BGP Routes
permalink: /guides/reject_invalids/
---

* TOC
{:toc}

# Reject RPKI Invalid BGP Routes

## Purpose

A complete filter for any BGP configuration should reject RPKI invalid BGP routes.

Whether a route is RPKI *invalid* or not is determined following [RFC 6811](https://tools.ietf.org/html/rfc6811) and [RFC 8893](https://tools.ietf.org/html/rfc8893).

It is considered *harmful* to manipulate BGP Path Attributes (for example LOCAL_PREF or COMMUNITY) based on the RPKI Origin Validation state.
Making BGP Path Attributes dependent on RPKI Validation states introduces needless brittleness in the global routing system as explained [here](https://mailarchive.ietf.org/arch/msg/sidrops/dwQi9lgYKRVctdlMAHhtgYkzhSM/).

# Configuration Examples

## OpenBGPD

This is an example how to do Origin Validation without the RPKI-To-Router protocol.
Ensure the `rpki-client` root crontab entry is not commented out, and runs every hour.

```
# crontab -l | grep rpki
~   *   *   *   *   -ns   rpki-client && bgpctl reload
```

Import the `rpki-client` generated config and instruct `bgpd` to reject RPKI invalid routes

```
include "/var/db/rpki-client/openbgpd" # consume VRPs from rpki-client
deny quick from ebgp ovs invalid       # dont import invalids
deny quick to ebgp ovs invalid         # dont export invalids
```

## BIRD

The [rtrsub](https://github.com/job/rtrsub) utility can be used to generate static ROA tables for BIRD 1.6.

Then the next step is to reference a `reject_invalids()` function in all EBGP import and export filters.

```
function reject_invalids()
{
  if (roa_check(roas, net, bgp_path.last) == ROA_INVALID) then {
    print "Reject: RPKI Invalid: ", net, " ", bgp_path;
    reject;
  }
}
```

## Junos

Configure RTR

```
routing-options {
  autonomous-system 64511;
  validation {
    group rpki-validator {
      session 10.1.1.6 {
        local-address 10.1.1.5;
      }
    }
  }
}
```

Instruct the router to reject RPKI invalid routes, and also mark `not-found` and `valid` routes with a non-transitive state.
*note*: BGP communities or other BGP Path Attributes *MUST NOT* be modified based on the validation state!

```
policy-statement rpki {
  term reject_invalid {
    from {
      protocol bgp;
        validation-database invalid;
    }
    then {
      validation-state invalid;
      reject;
    }
  }
  term mark_valid {
    from {
      protocol bgp;
      validation-database valid;
    }
    then {
      validation-state valid;
      next policy;
    }
  }
  then {
    validation-state unknown;
    next policy;
  }
}
```

## Cisco classic IOS and IOS XE

Configure RTR

```
# show running-config | begin bgp

router bgp 64500
 bgp log-neighbor-changes
 bgp rpki server tcp 10.1.1.6
```

Match and deny based on RPKI validation state.

```
route-map ebgp-in deny 1
 match rpki invalid
!
```

## Cisco IOS-XR

Configure RTR and enable RPKI for each address family
```
router bgp 1
 bgp bestpath origin-as allow invalid
 address-family ipv4 unicast
   bgp origin-as validation enable
 address-family ipv6 unicast
   bgp origin-as validation enable
 rpki server 192.1.0.2
```
*note*: In XR before 6.6.3, the default RPKI behavior is that validation is enabled in each AFI, and had to be specifically disabled if it was not wanted.  In XR 6.6.3 and later, the default behavior is disabled and validation must be enabled.  It is not possible to enter the enable commands in versions before 6.6.3, so any XR router that is upgraded to 6.6.3 will come up with validation disabled and the specific command lines above must be entered manually upon first bootup.

Match and deny based on RPKI validation state
```
route-policy rpki-validate
  if validation-state is invalid then
    drop
  endif
end-policy
```
Apply policy in policy chain where needed
```
route-policy ebgp-in
  apply rpki-validate
  <rest of policy>
end-policy
```
