# Native settings: VIctory rows in the title's own menu

VIctory adds its own settings to the title's **Settings → Graphics** tab as
real native rows (◀ ▶ lists), saved in `victory.toml`. This page explains how
the pieces fit and how to add or change a setting.

## The pieces

| Where | What | Touches the title? |
|---|---|---|
| `crates/victory-core/src/settings/` | The settings themselves: enums, labels, `victory.toml` spelling, checks, saving | No (pure, unit-tested) |
| `apps/victory/src/menu_trace/settings_list.rs` | The settings as the menu sees them: current choice, range, save | No (unit-tested) |
| `apps/victory/src/menu_trace/upscaling.rs` | The Upscaling section: heading and the rows it holds | Yes: calls two title helpers, reads the group's row list |
| `apps/victory/src/menu_trace/native_list.rs` | Turns a title row into a native ◀ ▶ row for one setting | Yes: rewrites title rows |

A setting is a **fieldless enum** with one `Choice` impl. Everything else
(the row's option count, its labels, reading the value at startup, writing it
when the player changes it) follows from that impl.

```text
victory.toml ──read──▶ Config ──from_config──▶ ListSetting ◀──▶ native row (◀ ▶)
     ▲                                             │
     └──────────────── file::save ◀──commit────────┘
```

## Adding a setting

Say we want **Frame generation: Off / AMD FSR**.

**1. The value, in `crates/victory-core/src/config.rs`.** Add the enum (serde
spelling in `snake_case`), a field on `Config`, and read it like
`graphics_upscaler`:

```rust
#[derive(Debug, Default, Clone, Copy, PartialEq, Eq, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum FrameGeneration {
    #[default]
    Off,
    AmdFsr,
}
```

**2. The setting, in `crates/victory-core/src/settings/graphics.rs`.**

```rust
impl Choice for FrameGeneration {
    const CONFIG: ConfigKey = ConfigKey { section: "graphics", key: "frame_generation" };
    const NAME: Text = Text::own("victory_frame_generation", "Frame generation");
    const HELP: Text = Text::own("victory_frame_generation_desc", "Inserts generated frames. Applies after a restart.");
    const OPTIONS: &'static [Opt<Self>] = &[
        Opt { value: FrameGeneration::Off, label: Text::own("victory_frame_generation_off", "Off"), toml: "off" },
        Opt { value: FrameGeneration::AmdFsr, label: Text::own("victory_frame_generation_amd_fsr", "AMD FSR"), toml: "amd_fsr" },
    ];

    fn from_config(config: &crate::Config) -> Self {
        config.frame_generation
    }
}
```

**3. Its tests, in the same file.** Copy the upscaler's three:

```rust
#[test]
fn frame_generation_is_well_formed() {
    assert_eq!(validate::<FrameGeneration>(), Ok(()));
}

#[test]
fn frame_generation_spellings_load_back() {
    assert_matches_serde::<FrameGeneration>();
}
```

and add a `..._lists_every_variant` test with a `match` over all variants
(it stops compiling when a variant is added, until the test, and `OPTIONS`,
are updated).

**4. The row, in `apps/victory/src/menu_trace/`.** In `settings_list.rs`:

```rust
pub(super) static FRAME_GENERATION: ListSetting = ListSetting::of::<FrameGeneration>();
static ALL: [&ListSetting; 2] = [&UPSCALER, &FRAME_GENERATION];
```

and in `upscaling.rs` (or a new section next to it):

```rust
const ROWS: [&ListSetting; 2] = [&UPSCALER, &FRAME_GENERATION];
```

**5. What it does.** Read `config.frame_generation` wherever the feature
starts up (the upscaler is read in `apps/victory/src/title_patch.rs`).
Settings apply after a restart: the menu only changes the saved value.

## Changing a setting

* **Rename the label or help:** change the English in `Text::own(...)`. Keep
  the key (the first argument) unless you want a new ID.
* **Add an option:** add the enum variant, its serde spelling, and an `Opt`.
  The tests above fail until all three agree.
* **Reorder options:** reorder `OPTIONS`. The saved value is the spelling,
  not the position, so existing `victory.toml` files keep their choice.

## What the checks catch

`validate::<T>()` (one test per setting) refuses:

* no options, or the same value / spelling / label ID twice;
* a `victory.toml` spelling that is not a plain name (`a-z`, `0-9`, `_`);
* a default value that is not among the options;
* a name and help with the same ID.

`assert_matches_serde::<T>()` refuses an option spelled differently from what
`victory.toml` loads, so a value saved from the menu always loads again.

`Text::own` refuses keys that do not start with `victory_` (at compile time
in a `const`). That keeps our keys apart from the title's by name; a 32-bit
hash can still collide, which the tests check among our own IDs.

Saving refuses a `victory.toml` it cannot edit cleanly (an inline
`graphics = { ... }` table, dotted keys): the edited text is parsed and
compared first. A failed save keeps the old choice.

## Saving

`settings::file::save` changes only the one line `[section] key = '...'`:
comments, order and other values in the user's `victory.toml` stay as they
are. A missing key goes under its section header; a missing section is
appended. The file is replaced in one rename, so a crash cannot leave half a
file.

## Known limits (prototype)

* **Option names.** The row borrows the title's quality list class, so the
  menu shows that list's option names (Low, Medium, …) instead of ours. Our
  labels (`Opt::label`) are ready for when the option text path is found.
* **Counter lags.** After ◀ ▶ the option text changes at once, but the
  `n / count` counter updates when the tab is shown again: the title only
  re-publishes a row after its "settings changed" event, which VIctory does
  not send yet. (Publishing from commit is not an option: the UI context the
  title passes to publish lives on its stack.)
* **Lists only.** Toggles and sliders use other native row kinds; they will
  get their own trait next to `Choice` once their rows are understood.
* **Behind a switch.** The section only appears with
  `VICTORY_MENU_UPSCALING_PROBE=1` (and `VICTORY_MENU_GRAPHICS=1`, which adds
  the hidden Graphics tab).

## Text IDs

The title looks every label up by a 32-bit `joaat` hash of a key. VIctory's
keys start with `victory_` (see `Text::own`). One title ID is used as-is
(`Text::title`): the Graphics tab's name. The tab bar shows the text of a
section's *screen* ID (Display's "Video" sits under its screen `c399a717`),
so the hidden Graphics tab's name goes under the screen VIctory registers it
with, `9a46cdb1`.
