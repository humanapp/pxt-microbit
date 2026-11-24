# The micro:bit simulator

The simulator is two TypeScript projects. `sim/core/` is the board state and the shims, and it runs
without a browser. `sim/visuals/` is the view, which draws the board and turns clicks into state
changes. The split follows the pxt-core design in `pxtsimcore/README.md`, which explains the
general pattern. This file covers how pxt-microbit applies it.

## Layout

- `sim/core/dalboardstate.ts` has `DalBoardState`, which builds the LED matrix, buttons, pins,
  radio, sensors and the other part states.
- `sim/core/boardaccess.ts` declares `DalBoard`, the board class the shims use, along with the
  `board()` and `boardView()` accessors.
- `sim/core/headless.ts` sets up a run with no view.
- The rest of `sim/core/` holds the state and shims for each area of the board, such as
  `ledmatrix.ts`, `music.ts` and `compass.ts`.
- `sim/visuals/dalboardview.ts` has `DalBoardView`, the view, and sets up each browser run.
- `sim/visuals/microbit.ts` draws the board, and files such as `buttonpairsim.ts`, `ledmatrix.ts`
  and `microservo.ts` draw its parts.

## Building

`sim/tsconfig.core.json` compiles `sim/core/` without the `dom` lib, so core code that uses
`document` or `window` fails to build. It compiles against `pxtsimcore.d.ts` and
`pxtsimcore-globals.d.ts` from pxt-core, and writes `built/simcore.js` and `built/simcore.d.ts`.

`sim/tsconfig.json` compiles `sim/visuals/` with the `dom` libs, against pxt-core's `pxtsim.d.ts`
and `built/simcore.d.ts`. It lists `../built/simcore.js` under `prepend`. `pxt buildtarget` builds
the core config first and this one second. It then puts `simcore.js` in front of the view code, so
`built/sim.js` holds both halves. The declarations are joined the same way in `built/sim.d.ts`.
`sim/public/simulator.html` loads pxt-core's `pxtsim.js` and then `sim.js`.

All of `sim/core/` is joined into one script, so a file that uses another file while the script
loads must come after it. A `/// <reference path>` pins that order. `boardaccess.ts` references
`dalboardstate.ts` because `DalBoard` extends `DalBoardState`. `headless.ts` references
`boardaccess.ts` for the same reason.

## pxt-common-packages files

The micro:bit simulator does not load `common-sim.js`. It compiles the pxt-common-packages files it
uses into its own bundles instead. The core config lists the DOM-free ones from each library's
`sim/core/` folder, such as the radio, settings and control message state. The view config takes
just one file, `libs/core/sim/neopixel.ts`. To use another common library in the simulator, add
its files to the matching config.

## Board state and view

`runtime.board` is the `DalBoard`. Shims and state code get it through `board()`.
`runtime.boardView` is the `DalBoardView`, and only view code uses it. View code gets it through
`boardView()`, which returns `undefined` in a headless run.

State code never calls the view. When it changes something the view shows, it calls
`runtime.queueDisplayUpdate()`, and the view redraws in `updateView`. For example,
`pins.setAudioPinEnabled(false)` marks the audio pin as disabled in the edge connector state and
queues an update. The view then draws a disabled-headphone image over the headphone part.

`DalBoardView` builds a `visuals.BoardHost` over the state in `initAsync`. It switches the board to
version 2 when the program uses a version 2 part, such as the microphone or the touch logo. On
Safari it also adds a mute button, which stays until the user unmutes.

Sound goes through `pxsim.audiohost` from pxt-core, so core code plays sound the same way with or
without a browser. Recording needs the browser, so core records through the `record.RecorderHost`
interface, whose default reports recording as unsupported. When the view bundle loads,
`sim/visuals/record-audio-dom.ts` calls `setRecorderHost` to install a MediaRecorder host.

## Starting a run

For a browser run, `initRuntimeWithDalBoard` in `dalboardview.ts` creates a `DalBoard` and a
`DalBoardView`. It is set as `pxsim.initCurrentRuntime`, unless something set that first. For a
headless run, `initHeadlessDalBoard` in `headless.ts` creates a `DalBoard` with no view. It is set
as `pxsim.initHeadlessRuntime` in the same way. Because `headless.ts` is in core, both `simcore.js`
and `sim.js` include it. In either kind of run, a runtime error draws a sad face on the LEDs.

## Running programs headless

The programs in `tests/sim/` run in Node with no browser. Each one lists the serial output it
expects in `// @serial:` comments, and the run fails if its output does not match. This one checks
that a click on button A runs its handler.

```ts
// @serial: A pressed
// @serial: shown
input.onButtonPressed(Button.A, function () {
    console.log("A pressed")
})
control.raiseEvent(DAL.MICROBIT_ID_BUTTON_A, DAL.MICROBIT_BUTTON_EVT_CLICK)
pause(50)
basic.showString("Hi")
console.log("shown")
```

After `pxt buildtarget`, run them from the repository root, as CI does after `pxt ci`.

    pxt simtest --base libs/blocksprj tests/sim

When `built/simcore.js` exists, `pxt simtest` loads it with pxt-core's `pxtsimcore.js`, so the run
uses no view code at all. A program that calls a shim defined only in `sim/visuals/` fails there.
The "Running programs headless" section of pxt-core's `pxtsimcore/README.md` explains the timing
and the other options.

## Open items

- Typed simulator inputs are a proposal, not code yet. A hidden API such as
  `sim.input.click(Button.A)` would let a program press buttons or set sensor values through the
  same code the view uses. Today a program raises DAL events directly, as in the example above.
- `input.magneticForce` always returns 0, because no state backs it yet.
