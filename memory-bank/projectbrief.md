# Project Brief

This project is an isometric city builder web application, originally implemented in vanilla JavaScript and later migrated to Vue 3 with Vite. The application allows users to place, remove, and label isometric tiles (buildings) on a grid, with persistent state stored in localStorage. The project is designed for rapid prototyping, user-friendly interaction, and extensibility.

## Core Requirements and Goals

- Users can build an isometric city by placing tiles on a grid.
- Users can select different tile types (from a texture atlas) and place them on the grid.
- Users can remove tiles via a context menu.
- Users can add and edit labels for each tile via the context menu.
- The city state (tiles and labels) is persisted in localStorage and restored on page load.
- The UI includes a "Kaart Wissen" (Clear Map) button to reset the city after confirmation.
- The application is implemented as a Vue 3 Single File Component (SFC) and uses Vite for development/build.
- The codebase is structured for maintainability and extensibility.

## Source of Truth

This document defines the core requirements, goals, and scope for the isometric city builder project. All other documentation files in the memory bank build upon this foundation.