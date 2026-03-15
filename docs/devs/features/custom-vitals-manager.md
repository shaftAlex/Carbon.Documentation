---
title: Custom Vitals Manager
description: Management support for Rust's Custom Vitals
---

# Custom Vitals Manager


<b>Custom Vitals Info</b> is a Facepunch class which defines a multitude of options to display on vitals above the player health area.

<image src="/misc/custom-vitals.webp"/>


## Introduction

The content structure is fairly straightforward. Theres a background color, two text objects, an icon and a countdown status value you can use. There currently isn't a health-bar like vital variant.

:::code-group
```cs [CustomVitalInfo]
public Color backgroundColor;
public Color leftTextColor;
public Color rightTextColor;
public Color iconColor;
public string leftText;
public string rightText;
public string icon; // CRC FileStorage value (uint)
public bool active;
public int timeLeft;
```
:::

### The Manager

There are two types of custom vitals that the manager... manages. The <b>Shared vitals</b>, and the <b>player</b>-specific <b>vitals</b>. 

Here's the [<b>source</b>](https://github.com/CarbonCommunity/Carbon/blob/develop/src/Carbon.Components/Carbon.Common/src/Carbon/Components/CustomVitalManager.cs) for the implementation.

## Shared Vitals

<b>Shared vitals</b> are vitals which will be delivered to all connected players (as well as newly connected ones).

:::code-group
```cs [Shared Vitals Sample]
using Carbon.Components;
using Carbon.Modules;

public readonly ImageDatabaseModule ImageDatabase = Carbon.Base.BaseModule.GetModule<ImageDatabaseModule>();

public CustomVitalManager.SharedIdentifiableVital globalVital;

private void OnServerInitialized()
{
    // Rent one from the manager's class or just use Pool.Get<CustomVitalInfo>()
    //   they're one and the same thing
    var vital = CustomVitalManager.RentVitalInfo(
        icon: ImageDatabase.GetImageString("file"), 
        iconColor: Color.red.WithAlpha(.6f), 
        backgroundColor: Color.black.WithAlpha(.9f), 
        leftText: "Searching files...", leftTextColor: Color.white.WithAlpha(.6f));

    // Add and immediately apply the shared vital to all players
    // expiry will automatically remove the global vital from all players 
    globalVital = CustomVitalManager.AddSharedVital(vital, expiry: 10);
}

private void Unload()
{
    // Remove the shared vital. Changes are immediately applied to all players.
    CustomVitalManager.RemoveSharedVital(globalVital);

    // When in doubt, clear ALL shared vitals ever created, ever. 
    CustomVitalManager.ClearSharedVitals();

    globalVital = null;
}
```
:::

:::warning NOTICE

When vitals are removed, they'll appropriately be sent back to Facepunches Pool.

:::

## Player Vitals

<b>Player vitals</b> are individual vitals, unique per player. 

<video controls autoplay loop src="/misc/sillygoosery.mp4"></video>


:::code-group
```cs [Player Vitals Sample]
using Carbon.Components;
using Carbon.Modules;

public readonly ImageDatabaseModule ImageDatabase = BaseModule.GetModule<ImageDatabaseModule>();

public CustomVitalManager.PlayerIdentifiableVital playerVital;

private void OnServerInitialized()
{
    var self = BasePlayer.Find("Raul");
    var vital = CustomVitalManager.RentVitalInfo(
        icon: ImageDatabase.GetImageString("reload"), 
        iconColor: Color.yellow.WithAlpha(.6f), 
        backgroundColor: Color.black.WithAlpha(.9f), 
        leftText: "Standing by...", leftTextColor: Color.white.WithAlpha(.6f));
        
    playerVital = CustomVitalManager.AddVital(self, vital);
}

private void Unload()
{
    var self = BasePlayer.Find("Raul");
    CustomVitalManager.ClearVitals(self); 
}

[ConsoleCommand("sillygoosery")]
private void sillygoosery(ConsoleSystem.Arg arg)
{
    var player = arg.Player();
    player.Ragdoll();
    playerVital.info.icon = ImageDatabase.GetImageString("star");
    playerVital.info.leftText = "Ragdolling";
    playerVital.info.backgroundColor = Color.red.WithAlpha(.6f);
    playerVital.info.rightText = "{timeleft:ss}s";
    playerVital.info.rightTextColor = Color.white;
    playerVital.SetTimeLeft(5); // Restarts the timer of the vital
    playerVital.SendUpdate();

    timer.In(5f, () =>
    {
        playerVital.info.icon = ImageDatabase.GetImageString("reload");
        playerVital.info.leftText = "Standing by...";
        playerVital.info.rightText = string.Empty;
        playerVital.info.backgroundColor = Color.black.WithAlpha(.9f);
        playerVital.SendUpdate();
    });
}
```
:::

## Updating Vitals

Both Shared and Player vitals follow the same format for applying and sending updates to the designated targets. Highly recommended to have a look at the example above on how changing vital variables works.

## TimeLeft Formatting

The `{timeleft:}` placeholder displays a countdown timer. With nothing after the colon, it defaults to `hh:mm:ss` format (e.g., `01:05:15`).

### Format Tokens

Each token is replaced with its corresponding time component. Double-letter tokens are zero-padded; single-letter tokens are not.

| Token | Description |
|-------|-------------|
| `s` | Seconds, no padding |
| `ss` | Seconds, zero-padded |
| `m` | Minutes, no padding |
| `mm` | Minutes, zero-padded |
| `h` | Hours, no padding |
| `hh` | Hours, zero-padded |
| `d` | Days, no padding |
| `dd`, `ddd`, ... | Days, padded to token width |

Each component displays only its own unit- seconds up to 60, minutes up to 60, hours up to 24. Days can extend into the thousands.

### Inserting Literal Characters

Use `\\` to escape each literal character. For example, `\\:` inserts a colon and `\\m` inserts the letter "m". Spaces also need to be escaped.

### Examples

Using a remaining time of 1 day, 3 hours, 5 minutes, and 7 seconds

| Format String | Output |
|---------------|--------|
| `{timeleft:}` | `03:05:07` |
| `{timeleft:hh\\:mm\\:ss}` | `03:05:07` |
| `{timeleft:h\\:mm\\:ss}` | `3:05:07` |
| `{timeleft:mm\\:ss}` | `05:07` |
| `{timeleft:m\\:ss}` | `5:07` |
| `{timeleft:ss}` | `07` |
| `{timeleft:s}` | `7` |
| `{timeleft:m\\m\\ s\\s}` | `5m 7s`|
| `{timeleft:h\\h\\ m\\m\\ s\\s}` | `3h 5m 7s` |
| `{timeleft:d\\d\\ h\\h\\ m\\m\\ s\\s}` | `1d 3h 5m 7s` |

:::info
Placing tokens directly adjacent without an escaped separator (e.g., `ms`, `mms`) can cause unpredictable display behavior.
Using multiple `{timeleft:}` placeholders in the same string (e.g., `{timeleft:mm}:{timeleft:ss}`) breaks the formatting. Use a single placeholder with escaped separators instead.

Formatting list and info curated by [shaftAlex](https://github.com/shaftAlex).
:::
