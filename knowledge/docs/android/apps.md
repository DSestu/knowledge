# Apps

## Unlisted apps

* [Kotatsu](https://github.com/KotatsuApp/Kotatsu): manga reader (scans, offline etc)

* [Gamenative](https://github.com/utkarshdalal/GameNative): Playing steam games on android. Connect your steam account, local emulation.

* [Aurora Store](https://auroraoss.com/): alternative app store with only free and open source apps.

## Master of olympus on Gamenative

To play Master of Olympus, you need to:

1. Install the game from gamenative.

**Fix the non visible text and menus:**

1. Download <https://github.com/FunkyFr3sh/cnc-ddraw/releases> zip file.

2. Extract it in the folder of the game.

3. Use the resolution of the default game (800x600).

I didn't test for the resolution patch. But I think you should adapt then the screen resolution.

**Fix the sound:**

1. Container options > Composants win > Directsound.

2. Change `Native (Windows)` to `Builtin (Wine)`.

**Touchscreen:**

1. Container options > Controller > Mode touchscreen, enable it. In it's own settings, activate the double finger right click.

Be careful, the double finger is: Left finger first without move, then second finger.
