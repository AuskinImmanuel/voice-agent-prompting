# Location mapping and human handoff for multi-branch voice agents

One phone number, seven branches, and a caller who opens with "I need to bring my car in." Nothing there names a branch, and the caller isn't choosing one. They think they're calling the dealership.

Multi-location routing looks small on a whiteboard and becomes a large part of the build. Three tables sit under it and have to agree: locality to branch, branch plus capability to hand-off branch, and destination to transfer rule. The example is an invented Pune dealership group that sells and services cars, plus one showroom it doesn't own.

```
Branch        Code      Bays  Notes
Kothrud       PNQ-KOT   0     flagship, sales only
Baner         PNQ-BAN   0
Viman Nagar   PNQ-VMN   4     paint booth
Kharadi       PNQ-KHD   6     service only, the EV bay
Hadapsar      PNQ-HAD   2
Katraj        PNQ-KAT   0
Wakad         PNQ-WAK   0     open Thursday to Saturday
Nashik        PARTNER   n/a   not ours. Transfer only.
```

---

## Derive the branch, don't ask for it

Never ask the location question while the answer is already in the room.

```
Resolution order (hard):
1. The caller's own words. A named branch wins. Stop.
2. Their record: most recent relevant visit. Skip detailing and pickup-drop,
   which follow the driver, not the car.
3. An address given on this call. New callers only.
4. At most one question. "Which side of town is easiest for you?"

Banned source: the address on the finance record. That is the paperwork's address,
not the caller's.

A car already in a bay at PNQ-VMN beats "nearest" for that repair.
```

Then confirm in one light line: "Sure. Viman Nagar, same as your last service?" Reading the list out loud turns the agent into a phone tree.

---

## Measured fallbacks, not list order

The common first-build failure: the fallback is whatever branch sits next in the table, and table order has nothing to do with driving. Give each locality an ordered ladder carrying real distance.

```
Caller locality  -> ordered ladder (road km)
Kothrud          -> PNQ-KOT 0, PNQ-BAN 6, PNQ-VMN 14, PNQ-KHD 16
Katraj           -> PNQ-KAT 0, PNQ-HAD 11, PNQ-KHD 24

1. Walk it in order. Offer the first branch that can do the job.
2. Skip a limited-hours branch unless the day fits. PNQ-WAK opens Thursday to
   Saturday; a Tuesday caller never hears about it.
3. Any rung over 20 km is said out loud with the number, and needs a yes.
```

A branch you name and then withdraw is worse than one you never named.

---

## Walling off the site you don't own

The Nashik showroom is a partner. The group can put a caller through and nothing else. That needs a wall, not a preference: a preference is what a model trades away the moment it looks helpful.

```
PARTNER, invariants:
- Never a rung on any ladder. Not a fallback for anybody.
- No branch of ours is offered to a caller who names it. The wall runs both ways.
- Nothing is written to our system here. No booking, no lead, no callback log.
- Named in the opening sentence: skip identification entirely.
- One clarifying question, only if it changes the destination:
    "A car you bought there, or a service booking?"
    bought -> partner_nashik_sales    service -> partner_nashik_service
```

Identifying a caller you can only transfer wastes their time: you check a name against a system with no authority over their question, then hand them to someone who asks again. This is the only place I let a caller skip identification, and it earns the exception.

---

## Capability tables are lookups, never inferences

A branch that can't do the job may still know one that can. That knowledge is a table, and only a table.

```
(branch, capability)  -> hand-off branch
PNQ-KOT + EV service  -> PNQ-KHD, EV bay, 16 km
PNQ-KAT + EV service  -> PNQ-KHD, EV bay, 24 km
PNQ-HAD + paint work  -> PNQ-VMN, paint booth, 12 km
```

A row exists: one offer, one destination, phrased as a question. The Katraj row becomes "The EV bay is Kharadi, about twenty-four kilometres from you. Should I check there?" Declined is closed.

No row: say nothing. Without it, a model reading a bays column infers that Kharadi, with six bays, can respray a bumper. It can't; the booth is at Viman Nagar. Inference over an infrastructure table produces confident, specific, wrong offers, and callers write those down.

---

## The transfer contract

Every human destination is a keyed rule with exact spoken anchor phrases. Matching happens on the anchor, not on meaning.

```
key:      svc_vimannagar_bay
anchors:  "our Viman Nagar service desk"
          "the service desk at Viman Nagar"

1. Speak one anchor, word for word, inside the farewell. Once per call.
2. Exact match only. "The Viman Nagar workshop" matches no rule, so it dials nothing.
3. Pass the key to the transfer tool character for character.
4. Phone numbers in the rules file are documentation. Route by key.
```

Rule 2 is the one that bites. Three rules carry the words Viman Nagar: `svc_vimannagar_bay`, `body_vimannagar_paint`, `sales_vimannagar_floor`. A natural goodbye reaches for "our Viman Nagar team," matching none of them. Copy, not compose, the discipline writeup 06 lands on for tool parameters.

File the follow-up task against the key, which cannot drift, and store the anchor beside it as a searchable field.

---

## When the dial fails

```
no rule matches  -> the fallback path. Never assemble a number from parts.
dial fails       -> one repaired retry, silent, then stop. No third attempt.
nobody picks up  -> the platform leaves the voicemail and files the follow-up,
                    unnarrated.
all three        -> one closing line: what was done, when someone calls back.
```

The repair depends on which failure came back. A rejected call means the request went out in the wrong shape. An empty response means `svc_vimannagar_bay` is not a key the rules file knows.

"I'm having a technical issue" is the weak version of that closing line. "They're not picking up right now. They have your number and the details, and someone will call you back today" ends the call properly.

---

## Warm transfer outward, structured payload inward

Two kinds of destination, two handoffs.

### External dial

The human is on a handset, mid-task, no screen, no context. Ringing that phone and dropping a stranger onto it is how transfers get refused. An outbound leg dials first, briefs them on who is calling and what they need, then bridges the caller on an explicit yes.

### Internal pool

Staff answer inside the platform and read a screen, so the handoff is a field set.

```
urgency      routine | same day | car not drivable
next_step    what the person picking up must do
reason       one line, in the caller's own words
branch       resolved branch for this call
vehicle      registration and model
caller_name  as given
callback     number to use
verified     yes | no
done_so_far  what the agent completed

Fill only from this call. Absent values get the literal string "not discussed".
Never infer one from the branch table or the caller's tone.
```

Urgency leads because "car not drivable" changes what the person picking up does before they read another field. `verified` is a field rather than an assumption, because they decide what to say next based on it.

Writeup 06 calls "not discussed" a bug, and it is, in a tool parameter: a backend takes it as data and writes it into a record, which is why the spec there asks for a server-side blacklist. Nothing parses this payload; a person reads it, and a blank field looks like an oversight while the sentinel says the question was never asked.

---

Multi-branch routing rewards being boring. Derive instead of asking, measure instead of ordering, wall off what you don't control, look things up instead of inferring them. Each swaps a judgment call for a table, the same trade writeup 01 makes for status codes.

The `/agents` folder of this repo carries what this layer sits on top of: appointment-type tables, a `phone_number` transfer node, and the escalation edges that reach it. None of the routing tables above are in there; those samples hold no location table and read the site list out of tool data. That is right at their scale, and stops being right around the third branch.
