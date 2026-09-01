# Pre-operative deterioration: can we see it coming at six hours?

## The task

> Six hours after a patient is admitted and booked for surgery, using only information available
> at that point, predict which of the patients still waiting are going to deteriorate before they
> reach theatre, within the 72 hours following admission. There is no label column in the data:
> you derive the outcome yourself from `events.csv`. Build a model, evaluate it honestly, and
> recommend an operating threshold you can defend, given that a false alarm has a real cost on a
> capacity-constrained ward.
>
> The full context, the data, and what to deliver are below.

## Contents

- [The task](#the-task)
- [Background](#background)
- [The scenario](#the-scenario)
- [Why this is hard](#why-this-is-hard)
- [Key terms](#key-terms)
- [What you have been given](#what-you-have-been-given)
- [What we are asking you to produce](#what-we-are-asking-you-to-produce)
- [Deliverables](#deliverables)
- [On the use of coding agents](#on-the-use-of-coding-agents)
- [How we will assess this](#how-we-will-assess-this)

## Background

Patients admitted to hospital for surgery do not go straight to theatre. There is a wait, for a
free slot, for the anaesthetic workup, for whatever else is ahead of them on the list. Most of
that wait is uneventful. But some patients get worse while they are waiting, before anyone has
operated on them at all: they need emergency transfer to intensive care, they are started on
blood-pressure-support drugs, a rapid-response team is called, they are intubated, they arrest, or
they die. Clinically this is sometimes called "failure to rescue". The early, often subtle signs
of deterioration (a drifting heart rate, falling oxygen, rising lactate) were there, and nobody
acted on them before the patient collapsed.

This matters operationally as much as it matters clinically. A hospital that could reliably flag
this a few hours in advance could reorder its theatre list, or get a critical-care review in
before the crash rather than after. That is the capability this case study asks you to evaluate:
not whether it is *possible in principle*, but whether it is *foreseeable from the data a hospital
actually has*, and what it would take to trust it.

## The scenario

Treat the following as the operating context for the exercise: a six-practice provider network,
described in enough detail that you can reason about it the way you would reason about a real
client's data.

A patient arrives, is admitted, and is booked for a procedure in the same breath. Admission and
booking are decided together, by the same team, at the same moment. Then the patient waits:
sometimes a few hours, sometimes two or three days, depending on the list, the anaesthetic
workup, and everything else competing for the same slot. The median wait from admission to
knife-to-skin, across this network, is around twenty hours.

Most of those patients wait uneventfully and go to theatre. A minority do not. Under the
assumptions behind this scenario, deterioration while waiting happens to roughly one in eighteen
of the patients who are still waiting six hours after admission. Every one of those is a patient
who had already been identified as needing an operation, and who the system then failed to get to
theatre in time.

The operating question is: **six hours after admission, using only what is known at that point,
can we identify which waiting patients are going to deteriorate before they reach theatre?** Six
hours is not arbitrary: it is roughly when a first set of vitals trends and a first round of labs
are back, and it is early enough that the answer can still change what happens next, pulling a
patient forward, asking critical care to review them, or keeping a bed warm.

## Why this is hard

Theatre capacity is fixed and contended. There is no spare list. Moving one patient up means
moving another patient down, and the patient who slides down is also, by definition, someone who
needs an operation. An alert that fires on a patient who was never going to deteriorate is not
free: it costs somebody else their slot, and the ward team's attention, which is also finite. So
this is not a search for the best headline number. What is wanted is an honest account of what a
tool like this could and could not do on these wards, and at what operating point, if any, it
would be worth switching on. Working out how to reason about that balance, and telling us how you
reasoned about it, is part of the task.

## Key terms

A short glossary, so nothing here depends on a clinical background.

| Term | Meaning |
|---|---|
| **Peri-operative** | The whole period around a procedure: before, during and after. This exercise is about the *before* only, the wait between admission and surgery. |
| **T0** | Time zero. The reference timestamp everything else is measured from: here, the moment of admission and booking (`admit_ts`). |
| **Deterioration** | A patient's condition worsening acutely. In this exercise it is defined by six specific event types, see `events.csv`, not by a vague sense that someone "looked unwell." |
| **Intubation** | Placing a breathing tube, typically because a patient can no longer protect their own airway or breathe adequately unassisted. |
| **ASA class** | A standard 1-5 score anaesthetists assign before surgery, summarising how healthy or unwell a patient is overall (1 = healthy, 5 = moribund). It appears in `surgeries.csv`. |
| **Urgency** | How soon a case needs to happen: `emergency`, `urgent`, `expedited`, or `elective_inpatient` (least urgent), in `surgeries.csv`. |
| **Failure to rescue** | The clinical term for missing the early warning signs of deterioration until the patient has already collapsed. It is the phenomenon this exercise is about detecting earlier. |

## What you have been given

Everything is in `data/`. One row per patient in `surgeries.csv`; `patient_id` is unique and one
patient corresponds to one booked procedure. 100,000 patients across 36 months and six practices.

| File | Contents |
|---|---|
| `surgeries.csv` | One row per patient: booking and theatre timestamps, procedure, urgency, demographics, ASA class, comorbidity flags, practice, and administrative fields. |
| `observations.parquet` | Long-format vitals and labs (`patient_id`, `surgery_id`, `ts`, `variable`, `value`, `unit`). Vitals are charted every two hours, labs roughly every six. 13.8M rows. |
| `medications.csv` | Timestamped inpatient administrations with drug name, class, dose and route. |
| `events.csv` | Timestamped clinical events during the inpatient wait. This is where the outcome lives. |
| `prior_history.csv` | Outpatient labs, ED visits, prior admissions, prior procedures and prior deterioration episodes from the three months before admission. |
| `data_dictionary.md` | Full column-by-column detail for all of the above. Read this before you start. |

Four things about the timing, which matter more than anything else here:

- **T0 is `admit_ts`.** Admission and booking are the same moment, so there is one clock, not two.
- **The prediction is made at T0 + 6 hours.** That is the decision point we care about.
- **Only information available at or before that moment may be used.** The tool would run at six
  hours on a live ward. It cannot know anything the ward did not know at six hours.
- **Everything is scoped to 72 hours from T0.** Beyond that we are not asking.

All timestamps are UTC. The intra-operative period is not in this dataset at all: no anaesthetic
record, no waveforms, because the question is entirely about the pre-operative wait.

## What we are asking you to produce

There is **no label column anywhere in this data**. We have not pre-computed the outcome for you,
and we would rather you did not assume ours. You derive it yourself from `events.csv`, and the
definition you land on is your decision to make and to justify: which events count, over what
window, measured from what point, and who is in the denominator.

The event types recorded are:

- `emergency_icu_transfer`
- `vasopressor_initiation`
- `rapid_response_activation`
- `intubation`
- `cardiac_arrest`
- `death`

Some deteriorating patients have several of these in sequence. Patients who had an uneventful
wait carry a `normal` row instead.

From there: build something that produces a risk estimate at T0 + 6 hours for a patient still
waiting, evaluate it in a way you would be willing to defend, and tell us what you found,
including the parts that did not work, or that told you something inconvenient.

## Deliverables

You have **48 hours** from receiving this repository. We expect roughly **8 to 10 hours of actual
work** and we would rather you stopped at that than pushed past it. Three things:

1. **A compressed archive of your code.** Runnable, with whatever instructions it needs.
2. **A slide deck** with your visuals and findings.
3. **A document** covering your process, your findings, and your call-outs.

For the deck and the document, we care about the reasoning and the judgment calls, not the volume.
A tight ten slides beats forty. In the document, please explicitly call out:

- the assumptions you made, particularly about the cohort and the outcome definition;
- anything you looked at and chose to exclude, and why you excluded it;
- and anything you would refuse to put in front of a clinician, and what would have to change
  before you would.

That last point is not a formality. It is one of the more informative things you can tell us.

## On the use of coding agents

Use them! We do, most of our team does, and we have no interest in pretending otherwise. This is
how people build things now, and being good at it is a real skill we are happy to see. We ask one
thing in return: transparency. Tell us where you used an agent and what for, in a paragraph or a
short list. That is all.

The friendly reminder is about the write-up and the deck: those need to be yours. An agent can
write the code, run the experiments and draw the charts, and that is fine. But the judgment calls,
the things you decided not to do, the assumptions you were uneasy about, and the argument you
would actually make to a clinician standing in front of you are the parts we are hiring for, and
they are the parts we will spend the live session on. Please do not simply ask an agent to
generate the report. We can tell, and it wastes the most interesting part of your submission.

## How we will assess this

In order of weight:

1. **How you framed the problem, and what you were sceptical about.** What question you decided
   you were actually answering, what you checked before trusting anything, and where you pushed
   back on the framing above.
2. **The rigour of the modelling.** Whether the evaluation supports the claims, whether the
   numbers mean what you say they mean, and whether someone could reproduce them.
3. **How well you communicate it** to a mixed clinical and operational audience: consultants,
   theatre managers, and a service director who will not read your code.

Afterwards, if selected for the next round, there will be a **45 to 60 minute live session** where
you walk us through your decisions. It is a conversation, not a presentation: expect us to ask why
you did things one way rather than another.

It's time to run on intelligence - good luck!
