# Formula Vision v2: Product UI Reference Notes

Formula Vision should feel like a motorsport instrument first and a website
second. The goal is not to reproduce another product pixel-for-pixel. It is to
borrow interaction patterns that already work well in racing software, then
express them with Formula Vision's own data model and visual system.

## Researched patterns

### Gran Turismo 7: pre-race Quick Menu

Gran Turismo 7's race flow uses a dedicated Quick Menu before entering a race.
Race information, track conditions and the start action are presented as
session-specific controls instead of generic website navigation.

Formula Vision application:

- keep the pre-race screen as a deliberate launch state;
- make session context visible before playback starts;
- use large, finite actions instead of generic menus;
- treat "start replay" as entering a race session rather than opening a page.

Reference:
https://www.gran-turismo.com/us/gt7/manual/race/01

### iRacing: Black Boxes and standalone widgets

iRacing's modern Sim UI separates standings, relative timing, lap timing and
weather into focused widgets. The 2026 UI work added standalone widgets,
minimal/expanded templates, per-widget scale/opacity controls and a Widget
Editor. Later updates added Pinned Mode so useful telemetry can stay visible
while the rest of the HUD is hidden.

Formula Vision application:

- Timing Tower, Driver Focus, weather and future Strategy/Battle views should
  remain composable analytical surfaces;
- dense panels should have clear compact/expanded states before adding more
  columns;
- future Focus Mode can pin Driver/Battle telemetry while hiding secondary
  chrome;
- keyboard access should remain a first-class interaction path;
- future customization should operate on a small set of stable widgets, not a
  free-form dashboard builder.

References:
https://www.iracing.com/iracing-development-update-november-2025/
https://www.iracing.com/iracing-development-update-august-2026/

### F1 TV: synchronized race information

F1 TV exposes live timing, telemetry, tyre usage, driver maps, onboards and
team radio as different views over one race session. Formula 1 also previously
used the product name **Battle Mode** for a two-driver same-screen comparison
experience.

Formula Vision application:

- keep a single authoritative replay clock;
- all analytical surfaces should seek/follow that same clock;
- the differentiator is historical replay plus analysis, not duplicating video;
- matchup and strategy experiences should feel like alternate race lenses,
  not separate pages;
- **do not ship Formula Vision's matchup feature under the name Battle Mode**.
  That name has already been used by F1 TV. Use Formula Vision's own mode
  identity instead.

Current working naming direction:

    RACE LENSES
      TRACK
      DUEL
      STRATEGY

This creates a game-like finite mode selector without cloning F1 TV's branding
or turning the product into a generic website tab bar.

References:
https://www.formula1.com/en-us/subscribe-to-f1-tv
https://www.formula1.com/en/latest/article/how-to-make-the-most-of-f1-tv-pro.iIaRABNtdWkGTOHaJV9wT.iIaRABNtdWkGTOHaJV9wT

## Formula Vision design rules

1. **Session, not page.** Once a replay is open, navigation should behave like
   an in-game session shell. Global website links recede.
2. **Finite modes beat giant menus.** Race Select, Results, Driver Focus,
   Battle and Strategy should be explicit modes/actions.
3. **Information can float; truth cannot.** Panels can move or collapse later,
   but timing/order/gaps always come from canonical race products.
4. **Controller-like clarity.** Primary actions get compact key/button hints and
   obvious selected states.
5. **Minimal and expanded states.** Dense widgets should grow intentionally
   rather than showing every metric at all times.
6. **Track-first composition.** The spatial replay remains the visual center.
7. **No fake instrumentation.** Decorative game language is acceptable; fake
   race metrics, loading percentages and inferred timing are not.
8. **Motion is feedback.** Use short transitions for selection, mode changes and
   event arrival. Do not animate data just to make the screen look busy.
9. **Own the implementation.** Do not copy proprietary game/company source
   code. Recreate useful interaction patterns with Formula Vision components.
10. **Open-source reuse only with license review.** If a third-party component
    is introduced later, record its package, license and attribution before
    shipping it.

## Near-term visual backlog

These are product-polish items, not race-correctness blockers:

- tune CHASE camera smoothness while preserving the decoupled vehicle/guide model;
- richer start/finish and pit-lane rendering where source geometry supports it;
- optional HUD Focus Mode;
- game-style Race Select / Session Menu overlay;
- minimal/expanded Timing Tower templates;
- DUEL/Strategy mode transitions;
- controller-friendly navigation if Formula Vision later targets TV/console
  browsers.

Shared circuit intelligence, zoom-aware MAP corner labels, and contextual CHASE
turn callouts are implemented.

The current shell already has the structural pieces these features need:
catalog-driven race selection, pre-race launch state, timing tower, central
track workspace, contextual driver panel, semantic event timeline and keyboard
controls.
