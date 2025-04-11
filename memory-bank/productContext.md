# Product Context

## Why This Project Exists

The isometric city builder was created to provide a simple, interactive, and visually appealing way for users to design and experiment with isometric city layouts directly in the browser. It serves as a playground for city planning, prototyping, and creative expression.

## Problems It Solves

- Provides a low-barrier, no-installation-required tool for isometric city design.
- Enables users to experiment with tile-based layouts and labeling without technical knowledge.
- Offers persistent state (via localStorage) so users can return to their city later.
- Supports rapid prototyping and iteration for game designers, hobbyists, and educators.

## How It Should Work

- Users select a tile type from a palette and place it on the grid.
- Users can right-click a tile to open a context menu with options to remove the tile or add/edit a label.
- Labels are displayed above the corresponding tiles for easy identification.
- The city state is automatically saved and restored using localStorage.
- A "Kaart Wissen" (Clear Map) button allows users to reset the city after confirmation.

## User Experience Goals

- Immediate visual feedback for all actions (placing, removing, labeling).
- Intuitive, minimal UI with clear affordances for all actions.
- Fast load times and smooth interaction, even on low-powered devices.
- Dutch language support for all user-facing text.
- No login or account required; all data is stored locally.