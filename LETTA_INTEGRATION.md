# Letta Integration for Voices of the Court

This document explains the new Letta integration features added to the VOTC mod.

## Overview

Letta (formerly MemGPT) integration allows characters to have persistent, stateful AI agents with managed memory systems. This integration adds new clipboard events to support:

- Creating Letta agents for characters
- Sending game events to agents
- Tracking save load/close for agent lifecycle management

## New Character Interactions

### Create Letta Agent

**Interaction ID:** `create_letta_agent_interaction`

Designates a character to have a Letta agent created for them. Once created, conversations with this character will use the Letta agent instead of traditional VOTC flow.

**Usage:** Right-click on a character → "Create Letta Agent"

**Clipboard Event:** `VOTC:AGENT_CREATE:[character_id]:[character_name]`

## New Scripted Effects

### send_letta_event

Sends a game event to a Letta agent. The event is queued and batched for efficiency.

**Scope:** Character who should receive the event

**Parameters:**
- `event_type` (string): Type of event (e.g., "birth", "death", "war_started", "title_gained")
- `event_desc` (string): Human-readable description of what happened

**Clipboard Event:** `VOTC:EVENT:[character_id]:[event_type]:[event_description]`

**Example Usage:**

```
# When a child is born
on_birth = {
    events = {
        my_mod.0001
    }
}

my_mod.0001 = {
    type = character_event

    immediate = {
        # Notify the mother
        mother = {
            send_letta_event = {
                event_type = "birth"
                event_desc = "Your [Select_CString(root.IsMale, 'son', 'daughter')] [root.GetTitledFirstName] was born"
            }
        }

        # Notify the father
        if = {
            limit = { exists = father }
            father = {
                send_letta_event = {
                    event_type = "birth"
                    event_desc = "Your [Select_CString(root.IsMale, 'son', 'daughter')] [root.GetTitledFirstName] was born"
                }
            }
        }
    }
}
```

### notify_letta_save_load

Notifies VOTC that a save has been loaded. Should be called from `on_game_start_after_lobby`.

**Scope:** Any

**Parameters:**
- `save_id` (string): Unique identifier for this save (use save file name or generate hash)
- `save_name` (string, optional): Display name for the save

**Clipboard Event:** `VOTC:SAVE_LOAD:[save_id]:[save_name]`

**Example Usage:**

```
on_game_start_after_lobby = {
    on_actions = {
        letta_save_load.0001
    }
}

letta_save_load.0001 = {
    type = empty

    immediate = {
        notify_letta_save_load = {
            save_id = "[GetCurrentDate.GetStringShort]_[GetPlayer.GetID]"
            save_name = "[GetPlayer.GetPrimaryTitle.GetNameNoTooltip] Campaign"
        }
    }
}
```

### notify_letta_save_close

Notifies VOTC that a save is closing (game exit or loading different save). Triggers backup of all agents.

**Scope:** Any

**Parameters:**
- `save_id` (string): Unique identifier for this save

**Clipboard Event:** `VOTC:SAVE_CLOSE:[save_id]`

**Example Usage:**

```
# Called when exiting to main menu or closing game
on_game_end = {
    on_actions = {
        letta_save_close.0001
    }
}

letta_save_close.0001 = {
    type = empty

    immediate = {
        notify_letta_save_close = {
            save_id = "[GetCurrentDate.GetStringShort]_[GetPlayer.GetID]"
        }
    }
}
```

## Recommended Event Types

Here are suggested event types to implement for a rich agent experience:

### Family Events
- `birth` - Child born
- `death` - Family member died
- `marriage` - Character got married
- `divorce` - Character got divorced

### Political Events
- `title_gained` - Gained a title
- `title_lost` - Lost a title
- `war_started` - War declared
- `war_won` - Won a war
- `war_lost` - Lost a war
- `imprisoned` - Character imprisoned
- `released` - Character released from prison

### Personal Events
- `injured` - Character wounded
- `recovered` - Recovered from illness/injury
- `trait_gained` - Gained a personality trait
- `trait_lost` - Lost a trait
- `became_rival` - Became rivals with someone
- `became_friend` - Became friends with someone

### Social Events
- `feast_hosted` - Hosted a feast
- `feast_attended` - Attended a feast
- `gift_received` - Received a gift
- `affair_started` - Secret affair began

## Implementation Checklist

To fully implement Letta integration in your mod:

- [ ] Add `on_game_start_after_lobby` event to call `notify_letta_save_load`
- [ ] Add `on_game_end` event to call `notify_letta_save_close`
- [ ] Add `send_letta_event` calls for major life events (birth, death, marriage)
- [ ] Add `send_letta_event` calls for political events (wars, titles)
- [ ] Test the "Create Letta Agent" interaction
- [ ] Ensure clipboard events are being copied correctly

## Technical Notes

- All clipboard events use the format: `VOTC:EVENT_TYPE:param1:param2:...`
- The `:` (colon) delimiter is used to separate parameters
- **Note:** Debug log entries use `/;/` delimiter, but clipboard events use `:` delimiter
- Events are batched and flushed based on:
  - Batch size (default: 10 events)
  - Time timeout (default: 5 minutes)
  - Conversation start (always flushes before conversation)
- Agents are isolated per save file - different saves have different agent universes

## Debugging

To verify events are being sent:

1. Enable CK3 debug logging
2. Check `logs/debug.log` for lines starting with `VOTC:AGENT_CREATE`, `VOTC:EVENT`, etc. (these use `/;/` delimiter)
3. Check VOTC application logs at `~/.config/Electron/votc_data/logs/debug.log`
4. Verify clipboard contains the expected event data after triggering (clipboard uses `:` delimiter)

## Example: Complete Birth Event Integration

```
namespace = letta_birth

# Birth notification to parents
letta_birth.0001 = {
    type = character_event
    hidden = yes

    trigger = {
        # Triggered on_birth
    }

    immediate = {
        # Notify mother
        if = {
            limit = { exists = mother }
            mother = {
                send_letta_event = {
                    event_type = "birth"
                    event_desc = "You gave birth to [Select_CString(root.IsMale, 'a son', 'a daughter')], [root.GetTitledFirstName]"
                }
            }
        }

        # Notify father
        if = {
            limit = { exists = father }
            father = {
                send_letta_event = {
                    event_type = "birth"
                    event_desc = "Your [Select_CString(mother.GetCharacter, mother.GetCharacter.GetTitledFirstName, 'wife')] gave birth to [Select_CString(root.IsMale, 'your son', 'your daughter')] [root.GetTitledFirstName]"
                }
            }
        }

        # Notify grandparents, siblings, etc. as desired
    }
}
```

## Support

For questions or issues with Letta integration:
- VOTC Application Repository: https://github.com/tanner-caffrey/Letta_VOTC
- VOTC Mod Repository: https://github.com/tanner-caffrey/Letta_VOTC_mod
- Original VOTC Discord: https://discord.gg/5nuE54GFgr
