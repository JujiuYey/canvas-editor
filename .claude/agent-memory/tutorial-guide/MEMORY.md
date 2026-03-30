# Tutorial Guide Agent Memory

## Project Overview
- Canvas-based rich text editor (not DOM-based like Quill/Tiptap)
- Main entry: `src/editor/index.ts` - Editor class
- Core engine: `src/editor/core/draw/Draw.ts` (~96KB, ~2800 lines)
- Build: UMD + ESM via Vite

## Architecture Summary
- Editor orchestrates subsystems via Command pattern
- Draw handles all canvas rendering
- Command -> CommandAdapt -> Draw bridge
- 40+ Particle types for content rendering
- 3 communication modes: listener (callbacks), eventBus (pub/sub), override (hooks)

## Tutorial Files
- 01-project-overview.md - Complete
- 02-main-entrypoint.md - Complete (main.ts Demo app)
- 03-editor-class.md - Complete (Editor class)
- 04-rendering-engine.md - Complete (Draw engine)

## Learning Path
1. Project overview
2. Main entry and Demo app
3. Editor class
4. Draw rendering engine (most complex)
5. Element system
6. Position & Range
7. Command system
8. Event system
9. Particle system
10. Plugin system

## User Context
- Learning canvas-based editor from scratch
- Limited rich-text-editor experience
- Basic TypeScript/frontend knowledge
