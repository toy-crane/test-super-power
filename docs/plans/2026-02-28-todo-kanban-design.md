# Todo Kanban Board - Design Document

## Overview

A general-purpose Todo web app with a Trello-style Kanban board UI. Supports task CRUD, tags, categories, due dates, priorities, subtasks, drag-and-drop, filtering/search, and dark mode. No backend — all data persisted in LocalStorage.

## Tech Stack

- **Framework:** Next.js 16 (App Router, React 19, TypeScript strict)
- **State Management:** zustand + persist middleware (LocalStorage)
- **Drag & Drop:** @dnd-kit/core + @dnd-kit/sortable
- **UI Components:** shadcn/ui (radix-nova) + Lucide React icons
- **Styling:** Tailwind CSS 4, oklch colors, CVA for variants

## Data Model

```typescript
type ColumnId = 'backlog' | 'todo' | 'in-progress' | 'review' | 'done';

interface Task {
  id: string;
  title: string;
  description?: string;
  status: ColumnId;
  priority: 'high' | 'medium' | 'low';
  tagIds: string[];
  dueDate?: string;       // ISO date string
  subtasks: Subtask[];
  order: number;
  createdAt: string;
  updatedAt: string;
}

interface Subtask {
  id: string;
  title: string;
  completed: boolean;
}

interface Tag {
  id: string;
  name: string;
  color: string;          // hex color
}

interface Column {
  id: ColumnId;
  title: string;
  taskIds: string[];      // ordered
}
```

Key decisions:
- Subtasks embedded in Task (1-level only, no separate normalization needed)
- Column `taskIds` array manages order within each column
- Tags stored separately, referenced by id from tasks

## State Management (zustand Store)

```
todoStore (zustand + persist middleware)
├── tasks: Record<string, Task>
├── columns: Record<ColumnId, Column>
├── tags: Record<string, Tag>
├── theme: 'light' | 'dark'
├── filters: { search, tagIds, priority, dueDateRange }
│
├── Task CRUD
│   ├── addTask(task)
│   ├── updateTask(id, partial)
│   ├── deleteTask(id)
│   ├── moveTask(taskId, fromCol, toCol, newIndex)
│   └── reorderTask(columnId, fromIndex, toIndex)
│
├── Subtask Actions
│   ├── addSubtask(taskId, title)
│   ├── toggleSubtask(taskId, subtaskId)
│   └── deleteSubtask(taskId, subtaskId)
│
├── Tag Actions
│   ├── createTag(name, color)
│   ├── updateTag(id, partial)
│   └── deleteTag(id)
│
├── Filter/Search Actions
│   ├── setSearchQuery(query)
│   ├── setFilterTags(tagIds)
│   ├── setFilterPriority(priority)
│   └── clearFilters()
│
└── Selectors
    ├── getFilteredTasksByColumn(columnId)
    └── getTasksByTag(tagId)
```

LocalStorage strategy:
- Persist entire store under `todo-app-storage` key
- Exclude filter state from persistence (reset on refresh)

## Component Architecture

```
app/page.tsx                    -- Main page (server component)
└── <BoardLayout>               -- "use client", full layout
    ├── <Header>                -- App title, dark mode toggle, search bar
    ├── <FilterBar>             -- Tag/priority/due date filter chips
    └── <KanbanBoard>           -- DndContext wrapper
        ├── <Column>            -- 5 columns (Droppable)
        │   ├── <ColumnHeader>  -- Column name + task count
        │   ├── <TaskCard>[]    -- Draggable cards
        │   │   ├── Title, priority badge, tag badges
        │   │   ├── Due date (highlighted when near)
        │   │   └── Subtask progress (2/5)
        │   └── <AddTaskButton> -- "+ Add" at column bottom
        └── <DragOverlay>       -- Ghost card during drag

<TaskDetailDialog>              -- Modal on task click
├── Title (inline edit)
├── Description (textarea)
├── Status (select)
├── Priority (select)
├── Tags (combobox, existing component)
├── Due date (date input)
├── Subtask list (checkbox + add/delete)
└── Delete button (alert-dialog, existing component)

<TagManageDialog>               -- Tag management modal
├── Tag list (color + name)
├── Add tag (name + color picker)
└── Delete tag
```

Existing shadcn/ui components to reuse: button, card, badge, input, textarea, select, combobox, alert-dialog, label, field.

Additional shadcn/ui components needed: dialog, popover, checkbox.

## Drag & Drop

- @dnd-kit/core + @dnd-kit/sortable
- Cross-column: drag task to another column changes `status` + inserts into target `taskIds`
- Within-column: drag up/down reorders within `taskIds`
- DragOverlay shows semi-transparent card preview
- Sensors: PointerSensor, TouchSensor, KeyboardSensor

## Interactions

Task creation flow:
1. Click "+ Add" at column bottom
2. Inline title input appears
3. Enter for quick create (defaults: column status, Medium priority)
4. Click card to edit details in modal

Filter/search:
- Search: text match on title/description (case-insensitive)
- Filters: tag, priority, due date range — AND logic
- Non-matching cards hidden, columns remain visible
- "Clear filters" button to reset all

## Visual Design

Dark mode:
- Toggle button in Header (Sun/Moon icons from Lucide)
- Uses existing `.dark` class oklch tokens in globals.css
- Store `theme` in zustand, toggle `dark` class on `<html>`
- Initial: detect `prefers-color-scheme`, then user choice takes priority

Priority badges:
- High: red, Medium: yellow, Low: blue

Due date display:
- Overdue: red text
- Today/tomorrow: orange text
- Other: default text

Subtask progress:
- "2/5" format on card + small progress bar

## Kanban Columns (Fixed)

| ID | Title | Description |
|----|-------|-------------|
| backlog | Backlog | Ideas and future tasks |
| todo | Todo | Ready to start |
| in-progress | In Progress | Currently working on |
| review | Review | Needs review |
| done | Done | Completed |
