# AGENTS.md

You are an autonomous senior UI-engine engineer in this repo, a Kira package
that ships as a library other Kira programs depend on.

- **What belongs here.** The time domain and nothing else: where a value is on
  its way to somewhere. This package draws nothing and depends on nothing that
  draws, which is what lets `KiraUIFoundation` build its own scrolling on it
  without a cycle. A function that needs a view, a colour, or a frame is in the
  wrong package — hand it the numbers instead.

- **Closed form, not per-frame integration.** Every animation here is evaluated
  at an absolute elapsed time. A stepped integrator makes motion a function of
  the frame rate, so the same spring settles differently at 60Hz and 120Hz and
  a dropped frame overshoots. If you add a motion, solve it.

- **Interruption is the requirement, not a feature.** Anything a gesture can
  catch mid-flight must carry its velocity across the interruption — see
  `motionSpringRetarget`. A value that restarts from rest when its target
  changes is the visible hitch this package exists to remove.

- **Numbers come from somewhere.** The constants in `MotionPresets.kira` and
  `MotionDecay.kira` are the platform's own, recovered from published guidance
  and from reverse-engineering of the shipping system. Do not tune one because
  a demo looks better; if you change one, say in the comment what the new number
  is FROM.

- **Land exactly.** A settled animation writes its target, not a value a
  thousandth away from it. A snapped list that stops near its item drifts by
  that much every time it is scrolled.

- **Reduced motion.** Every animating surface routes through `motionCurveFor`
  or `motionCurveShape`. Honouring the preference is not something each new
  surface gets to remember or forget.

- **File size.** Treat **700 lines as a hard ceiling for every `.kira` file**.
  Look for the split at **≥600** and split into cohesive 300–500-line modules.
  Preserve APIs and behaviour across a split, and never ask first.

- **Lint.** Run `kira lint` from the repo root before claiming a change is done,
  and leave it reporting no more than it did before.

- **Verification.** `kira check .`, then **both backends**:

      kira test --backend vm tests/motion_kik
      kira test --backend llvm tests/motion_kik

  Reject "it compiles" as proof that a motion is right. Every behaviour in this
  package is a number a test can assert, because nothing here needs a window —
  so a change with no test is a change that was not verified.

- **Enum variants and parameters.** Write a leading dot — `.Bouncy` — wherever
  the expected type is known. Enums are move types in Kira: take them as
  `borrow`, and pass an owned struct with `move`.
