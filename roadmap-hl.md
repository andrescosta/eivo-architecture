# Towards Alpha 1 release

## Large items with highest priority

- Eidos (1)
   - Simulation Framework based on Eidos primitives.
- Node discoverable for challenge, space, etc. (companion) 2 
- Integrate Examples, Exercises, Companions, and Eidos into Eibot by using Skills (3)
  - Refactor current instructions into a set of base system prompt and skills. The main objective is to create a set of skills to write eivolets. Another for the library, etc.
  - Updates tools examples with new discoverable node
- More Substrates onboarding using Eiva (5)
- Fill te catalog using Eibotron (6) and Eibot ??? Putting Eibot and Eibotron to create stuff
  5.1- Must include a test tool
- Documentation (7)
- Code reviews (Eidos, Runnerd)

## Technical or features Debts

- Add https://streamdown.ai/ instructions to crafts etc. (4)
- Playrooms reminders (8)
- Moveout from Zitadel to an ingress Oauth
- Budgets
- Overall smaller code improvements
- Use a different set of routes for discoverable capabilities(don't reuse)
  - We must check if challenge exists, before generating a new one
  - We must check if space exists, before generating a new one
- Discoverable UI design 
  - The discoverable needs a place in an eibook and they can show up at any level.
- Eibooklet main TOC needs a design.
- Ui overall:
  - Misaligned icons
- Github proces renames (new name for the runtime)
- Split lingv from Eivo
- New name for Eivo runtime and the Spec and bond together
- In certains ocasions, the namespace must not be part of the client initial URI but set by each call(for instance when we get a workbook). 
- Championship as discoverable
 
## Large items with lower priority

- Eibot as librarian

## Large items with no priority

Eibonator

## Agents roadmap

A tale of four robots:
- Eibot * Runtime componion
- Eibotron * Content generation 
- Eiva * Substrates onboarding
- Eibonator * Coordination


# Eidos roadmap

Found via real functional testing (EiKa/eivo-harness), not speculative design:

- Dynamic/live text — a `text` node's content is always a static declared string; no expression can drive it, so nothing can show a live-updating number (e.g. a running estimate).
- Eidos↔Anyma control bridge is fire-an-animation-only — a Frame Control/Button can't write simulation memory, call a handler, or adjust a generator's rate live, only replay a fixed pre-declared animation.
- `onMemory` handler-selection race for two handlers sharing the same event name — only safe today when routed through a channel round-trip.
- No string-valued `<Input>` type.
- `<Simulation>` layout can't nest (flat only).

# Playrooms roadmap

                     [ READER EXPERIENCE ]
                               │
       ┌───────────────────────┴───────────────────────┐
       ▼                                               ▼
[ In-Book Trigger ]                            [ Main Menu Hub ]
(In-the-flow & direct)                         (Management & hosting)
       │                                               │
       ├──► [ Primary ]: "Join Global Room"            ├──► Create New Room
       │                                               ├──► My Created Rooms
       └──► [ Secondary ]: Chapter Room List           └──► Full Global Directory
            • Select Public Room                            • Search / Filter all rooms
            • Select Protected Room (enter password)        • Direct join by password
