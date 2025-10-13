# 🎨 Axium Color Palette

_Axium: structure for your terminal._

This defines the official Axium colors used across the CLI, HUD, daemon output, and web interfaces.

---

## 🧭 Core Palette

| Role                                  | Color                                                         | Hex     | Notes                                                                         |
| ------------------------------------- | ------------------------------------------------------------- | ------- | ----------------------------------------------------------------------------- |
| **Primary (Teal)**              | ![#00B7C7](https://via.placeholder.com/20/00B7C7/000000?text=+) | #00B7C7 | Core Axium color — structure and balance. Used in HUD, logo, and highlights. |
| **Accent (Cyan)**               | ![#14D9E7](https://via.placeholder.com/20/14D9E7/000000?text=+) | #14D9E7 | Brighter accent for interactive elements, cursor glow, and palette focus.     |
| **Dark Graphite**               | ![#111111](https://via.placeholder.com/20/111111/000000?text=+) | #111111 | Main background for site, HUD, and daemon logs — terminal base tone.         |
| **Mid Graphite**                | ![#1B1B1B](https://via.placeholder.com/20/1B1B1B/000000?text=+) | #1B1B1B | Slightly lighter background for panels, code blocks, and secondary surfaces.  |
| **Soft White (Text)**           | ![#E6E6E6](https://via.placeholder.com/20/E6E6E6/000000?text=+) | #E6E6E6 | Default foreground text — high contrast but softer than pure white.          |
| **Muted Grey (Secondary Text)** | ![#A3A3A3](https://via.placeholder.com/20/A3A3A3/000000?text=+) | #A3A3A3 | Used for descriptions, metadata, or inactive HUD sections.                    |
| **Error / Warn Red**            | ![#FF4C4C](https://via.placeholder.com/20/FF4C4C/000000?text=+) | #FF4C4C | Errors, warnings, failed commands.                                            |
| **Success Green**               | ![#3ECF8E](https://via.placeholder.com/20/3ECF8E/000000?text=+) | #3ECF8E | Subtle success highlight in CLI confirmations or daemon OK states.            |

---

## 🧠 Usage Guidelines

### CLI / HUD / Daemon Output

- Primary (#00B7C7) → prefix, active env, key highlight
- Accent (#14D9E7) → dynamic or live feedback (palette selection, HUD blink)
- Muted Grey (#A3A3A3) → inactive items or status labels
- Error / Success → log output or CLI result messages

Example output (colors shown conceptually):

    [axium] env:prod  uptime:12m  status:ok
    axium → teal (#00B7C7)
    env → grey (#A3A3A3)
    prod → cyan (#14D9E7)
    ok → green (#3ECF8E)

---

### Web / Docs Theme

- Background: #111111
- Text: #E6E6E6
- Link / CTA: #00B7C7
- Hover / Focus: #14D9E7
- Code blocks: background #1B1B1B, text #14D9E7
- Logo / accents: blend of #00B7C7 → #14D9E7 gradient

---

### Logo & Favicon

- Base emblem color: #00B7C7
- Gradient accent: #14D9E7
- Background: transparent or dark graphite (#111111)

---

## 🧩 Tailwind Config Example

    // tailwind.config.js
    module.exports = {
      theme: {
        extend: {
          colors: {
            axium: {
              primary: "#00B7C7",
              accent:  "#14D9E7",
              dark:    "#111111",
              mid:     "#1B1B1B",
              text:    "#E6E6E6",
              muted:   "#A3A3A3",
              error:   "#FF4C4C",
              success: "#3ECF8E"
            }
          }
        }
      }
    };

Usage in HTML or JSX:

    `<div class="bg-axium-dark text-axium-text">`
      `<button class="text-axium-primary hover:text-axium-accent">`Run Axium`</button>`
    `</div>`

---

## ✅ Hex Summary

    Primary Teal:     #00B7C7
    Accent Cyan:      #14D9E7
    Dark Graphite:    #111111
    Mid Graphite:     #1B1B1B
    Soft White:       #E6E6E6
    Muted Grey:       #A3A3A3
    Error Red:        #FF4C4C
    Success Green:    #3ECF8E

---

_“Structure for your terminal.”_
