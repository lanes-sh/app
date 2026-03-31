# Labels & Filtering

> **Read before**: [The Issue Board](./issue-board.md)

Labels and filters help you organize a busy board. As you scale up to dozens of concurrent issues, color-coded labels and targeted filters keep everything findable.


## Creating Labels

There are two ways to create a label:

1. **From Settings** — Open Settings and navigate to the **Labels** tab. Type a name, pick a color, and click the add button.
2. **From the board** — Right-click any issue card, select **Add Label**, and use the inline label creator at the bottom of the picker. Type a name, choose a color, and confirm.

Labels have a maximum name length of 30 characters.


## Available Colors

Lanes offers 13 label colors:

| Color | Example use |
|-------|-------------|
| Red | Bugs, critical issues |
| Rose | Design tasks |
| Pink | UI work |
| Violet | Experiments |
| Indigo | Enhancements |
| Blue | Features |
| Sky | Documentation |
| Teal | Testing |
| Emerald | Performance |
| Amber | Warnings, tech debt |
| Stone | Infrastructure |
| Slate | Maintenance |
| Gray | Low priority, miscellaneous |

These are suggestions only. Use whatever color scheme works for your team or project.


## Assigning Labels to Issues

You can assign labels from two places:

- **Issue detail panel** — Select an issue on the board. In the detail panel, click the label area to open the label picker. Check one or more labels to assign them.
- **Right-click context menu** — Right-click an issue card on the board and select **Add Label**. The label picker appears as a popover where you can toggle labels on and off.

Issues can have multiple labels. Each label appears as a small colored badge on the issue card in the board view.


## Filtering by Labels

To filter the board to show only issues with specific labels:

1. Click the **filter icon** in the board header (near the sort controls).
2. Select one or more labels from the label filter dropdown.
3. The board immediately updates to show only issues that have at least one of the selected labels.

When a label filter is active, a **filter badge** appears in the board header showing the count of active filters. This makes it obvious when you are looking at a filtered view rather than the full board.


## Filtering by Working Directory

You can also filter the board by working directory:

1. Open the filter controls in the board header.
2. Select one or more working directories from the directory filter.
3. Only issues with matching working directories are shown.

This is useful when you have issues spread across multiple repositories or project folders.


## Filtering by Workflow Step

The step filter lets you show or hide issues based on their current column:

1. Open the filter controls.
2. Check or uncheck workflow steps (Planning, Implementation, Review, Done, etc.).
3. The board shows only issues in the selected steps.

By default, Done issues are filtered out to keep the board focused on active work.


## The Filter Badge

When any filters are active, a badge appears next to the filter icon in the board header. The badge shows the total number of active filter criteria. For example, if you have two labels selected and one directory selected, the badge shows "3".

This is your reminder that the board is not showing everything.


## Clearing All Filters

To reset the board to its unfiltered state, click the **Clear all** button in the filter panel. This removes all label, directory, and step filters at once, restoring the full board view.


## Combining Filters

Filters stack together:

- **Within a category** (e.g., multiple labels), filters use OR logic — an issue matches if it has any of the selected labels.
- **Across categories** (e.g., labels + directories), filters use AND logic — an issue must match at least one label AND at least one directory to appear.

For example, selecting the "Bug" and "Feature" labels plus a specific working directory shows all bugs and features in that directory.


## Sort Order

Sort order works independently of filters. You can sort the filtered results by:

| Sort | Behavior |
|------|----------|
| **Default** | Manual drag-and-drop order |
| **Newest** | Most recently created first |
| **Updated** | Most recently modified first |
| **Alphabetical** | A-Z by title |

Change the sort order from the sort control in the board header. The sort applies to all visible columns.

---

**Read next**: [Dependencies](./dependencies.md)
