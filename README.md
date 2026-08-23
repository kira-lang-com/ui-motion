# Kira UI Motion

Everything that moves: springs, scrolling, gestures that flex under a finger,
and things arriving and leaving.

The package owns the **time domain and nothing else**. It draws nothing, reads
no window, and depends on no package that does — a fling is a function of a
velocity and a clock. That is what lets `KiraUIFoundation` build its own
scrolling on top of it, and what lets every behaviour in it be tested with no
window open.

```
KiraLayout
  └─ KiraUIMotion        ← here
       └─ KiraUIFoundation
            └─ KiraUI
                 └─ OpacityUI
```

## What is in it

| | |
|---|---|
| `Core/MotionSpring` | A damped harmonic oscillator, solved rather than stepped. Under-, critically- and over-damped, with velocity-preserving retargeting. |
| `Core/MotionPresets` | The platform's own springs by name, reduced-motion handling, and easings. |
| `Core/MotionDecay` | Exponential fling decay, rubber banding and its inverse, snapping. |
| `Scroll/MotionScroll` | The scrolling state machine: drag → throw → edge → snap. |
| `Gesture/MotionPull` | Press a control and pull it: resistance, spring-back, and velocity-driven squash and stretch. |
| `Gesture/MotionGlow` | The light a touch energizes: a radial highlight that follows the finger and fades. |
| `Transition/MotionTransition` | Arriving, leaving, moving and morphing — all interruptible. |

## The two decisions everything else follows from

**Animations are solved, not stepped.** Each is evaluated at an absolute
elapsed time rather than integrated once per frame. A stepped spring is a
function of the frame rate: the same motion settles differently at 60Hz and
120Hz, and a late frame overshoots. Here a long frame lands exactly where three
short ones would have, and a test can ask where a spring will be at 300ms
without running eighteen frames to find out.

**Every motion carries its velocity across an interruption.** A list caught
mid-fling stops under the finger. A control told to leave while it is still
arriving leaves from where it actually is, at the speed it actually has. This
is what `motionSpringRetarget` is for, and it is the difference between an
interface that feels like an object and one that feels like a playback.

## Where the numbers come from

The constants are the platform's own rather than tuned by eye — recovered from
Apple's published motion guidance, from reverse-engineering of the shipping
system, and from the source of a shipped third-party implementation of the same
material. The tests assert those figures directly, so a curve that merely looked
similar would fail:

- **Rubber banding** — `(x·d·c)/(d + c·x)` with `c = 0.55`. Pulling 5 points
  against a 960-point dimension gives 2.74; pulling the whole 960 gives 340.65.
- **Fling decay** — 0.998 of the speed kept per millisecond (0.99 for surfaces
  that should not coast). A throw at 1000 points a second travels 499.5 points.
- **Springs** — `smooth` 0.5/1.0, `snappy` 0.5/0.85, `bouncy` 0.5/0.7, the
  default 0.55/0.825; repositioning 0.4/1.0, rotation 0.4/0.8, drawers 0.3/0.8,
  and 0.3/0.6 for glass under a finger.
- **Press and release are different springs** — down is 0.31s at damping 0.71,
  up is 0.41s at damping 0.36. The release is slower and bouncy enough to
  overshoot, and that asymmetry is most of what makes glass feel elastic. A
  press also *lifts* the element 20 points toward the finger rather than
  shrinking it.
- **The touch highlight** — a radial light at 0.1 opacity, 300 points across,
  centred on the finger and tracking it. In over 0.12s eased out, away over
  0.22s eased at both ends. Its falloff runs straight from full to half across
  the inner half of the radius, then eases to nothing at the rim.
- **Feedback** — a press acknowledges itself in about 100ms; a pointer may
  wander 10 points before it is a drag.

Glass **materializes** rather than fades: `MotionAppearance` carries a `lensing`
channel that lags the opacity, so the shape is legible before the material has
finished condensing. A host drawing plain surfaces can ignore it.

## Using it

Motion state is held by the caller and advanced once per frame. Nothing here
allocates, reads a clock, or keeps a global.

```kira
var scroll = MotionScrollState {}
let bounds = MotionScrollBounds { contentLength: 1000.0, viewportLength: 400.0 }

motionScrollDragBegin(scroll, pointerY, bounds)
motionScrollDragMove(scroll, pointerY, frameDelta, bounds)
motionScrollDragEnd(scroll, bounds, reducedMotion)

// Per frame. The answer is whether to ask for another one.
let moving = motionScrollAdvance(scroll, frameDelta, bounds, reducedMotion)
```

A pulled control answers with what to draw:

```kira
var pull = MotionPullState { reach: 120.0 }
motionPullPress(pull, x, y, reducedMotion)
motionPullMove(pull, x, y, frameDelta)
motionPullRelease(pull, reducedMotion)

let shape = motionPullAdvance(pull, frameDelta, reducedMotion)
// shape.x, shape.y, shape.scaleX, shape.scaleY, shape.pressScale, shape.intensity
```

## Building

```
kira check .
kira test --backend vm tests/motion_kik
kira test --backend llvm tests/motion_kik
kira lint
```

Requires a toolchain with the floating-point primitives this package is built
on — `exp`, `log`, `pow`, `min`, `max`, `round`, `hypot`, `fmod` and the rest.
A closed-form spring is `e^{-ζωt}` and a fling is `rate^t`; neither is
expressible without them.
