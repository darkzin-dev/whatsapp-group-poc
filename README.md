# WhatsApp Admin-Only Group PoC

> **Disclaimer:** This content is intended for research purposes only. I am not responsible for any account bans, restrictions, data loss, or other consequences that may result from its use. Proceed at your own risk.

## Overview

This Proof of Concept demonstrates unexpected behavior in the handling of a specific WhatsApp message type.

The message can be processed in an administrator-only group even when sent by a non-administrator participant.

## Reproduction

The behavior was investigated using runtime instrumentation of the WhatsApp iOS application.

The PoC has two main parts:

1. Changing the client-side administrator state.
2. Changing a value associated with `WASignalEncryptResult`.

An additional unicast event is then required to trigger the message-processing flow.

### 1. Administrator State

The first part changes how the iOS client sees the current user's administrator status.

The following methods are involved:

```objc
- (bool)currentUserIsAdmin
- (void)setCurrentUserIsAdmin:(bool)arg1
```

They are present in:

```text
WAChatSelectorViewController
WAMutableChatSession
WAStashedChatSession
```

The relevant code is:

```objc
%hook WAChatSelectorViewController

- (bool)currentUserIsAdmin {
    return YES;
}

- (void)setCurrentUserIsAdmin:(bool)arg1 {
    arg1 = YES;
    %orig;
}

%end

%hook WAMutableChatSession

- (bool)currentUserIsAdmin {
    return YES;
}

%end

%hook WAStashedChatSession

- (bool)currentUserIsAdmin {
    return YES;
}

%end
```

#### `WAChatSelectorViewController`

`currentUserIsAdmin` always returns `YES`.

This makes the local client consider the current user an administrator when this value is checked.

`setCurrentUserIsAdmin:` also forces the value passed to it to `YES` before the original method continues.

#### `WAMutableChatSession`

`currentUserIsAdmin` is also forced to return `YES` for the mutable chat session.

#### `WAStashedChatSession`

The same is done for the stashed chat session.

### What this changes

This only changes the **state reported by the local client**.

Because of that, administrator-only controls can become available in the UI, including options related to sending or editing messages that would normally be unavailable.

It does not actually promote the account to administrator on the server.

The interesting part comes from what happens after the client has been put into this state.

---

## 2. `WASignalEncryptResult`

The second part involves `WASignalEncryptResult`:

```objc
%hook WASignalEncryptResult

- (unsigned long long)version {
    return 3;
}

%end
```

The `version` method returns an unsigned 64-bit value. In the PoC, that value is changed to `3`.

The name `WASignalEncryptResult` suggests that the class is related to an encryption or signaling result inside the WhatsApp client.

WhatsApp relies on the Signal Protocol and other cryptographic components for its messaging system, so this class is part of that general area of the application.

The PoC does not establish exactly what the value `3` represents internally. What was observed is that changing this value is part of the setup needed for the message behavior described here.

For that reason, the value is documented simply as the `WASignalEncryptResult` value rather than assigning it a specific undocumented meaning.

---

## 3. Trigger Condition

Changing the administrator state and the `WASignalEncryptResult` value is not enough on its own.

An additional unicast event is needed to trigger the message flow.

A reaction is one example of an event that can trigger it.

After the event is processed, the affected message reaches the iOS client and is displayed as an `unknown` message.

---

## Observed Behavior

The message is processed in a group where only administrators are supposed to be able to send messages, even though the sending account is not actually an administrator.

On iOS, the resulting message appears as:

```text
unknown
```

The behavior was observed on WhatsApp for iOS.

The same behavior was not observed on WhatsApp Web during testing.

### Platform Behavior

| Client       | Result                                          |
| ------------ | ----------------------------------------------- |
| WhatsApp iOS | Message is processed and displayed as `unknown` |
| WhatsApp Web | Behavior not observed                           |

---

## How the Parts Fit Together

The administrator-related changes only affect what the local client believes about the user's role.

The `WASignalEncryptResult` change is part of the message-processing side of the PoC.

Once both are in place, the additional unicast event can trigger the flow that results in the `unknown` message being processed by the iOS client.

## Conditions

The behavior observed during testing required:

* An administrator-only group.
* The instrumented WhatsApp iOS client.
* The modified administrator state.
* `WASignalEncryptResult` returning `3`.
* An additional unicast event, such as a reaction.

These are the conditions under which the behavior was observed during testing.

## Credits

Found by **darkzin.dev** ( me ) in early 2022.

As of the last test, this behavior was still reproducible.

More:

* Website: https://darkzin.dev
* Discord: not builded yet
* GitHub: https://github.com/darkzin-dev

More research and projects are available on **darkzin.dev**.
