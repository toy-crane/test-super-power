# Todo Kanban Board Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a Trello-style Kanban board Todo app with task CRUD, tags, priorities, due dates, subtasks, drag-and-drop, filtering/search, and dark mode — all persisted in LocalStorage.

**Architecture:** Next.js 16 App Router with a single client-side `BoardLayout` component tree. zustand store with persist middleware handles all state and LocalStorage sync. @dnd-kit provides drag-and-drop between 5 fixed columns. Existing shadcn/ui (radix-nova) components are reused wherever possible.

**Tech Stack:** Next.js 16, React 19, TypeScript, zustand, @dnd-kit/core + @dnd-kit/sortable, nanoid, shadcn/ui (radix-nova), Tailwind CSS 4, Lucide React

**Design doc:** `docs/plans/2026-02-28-todo-kanban-design.md`

---

### Task 1: Install Dependencies

**Files:**
- Modify: `package.json`

**Step 1: Install runtime dependencies**

Run:
```bash
bun add zustand @dnd-kit/core @dnd-kit/sortable @dnd-kit/utilities nanoid
```

**Step 2: Add missing shadcn/ui components**

Run:
```bash
bunx shadcn@latest add dialog popover checkbox
```

**Step 3: Verify build**

Run: `bun run build`
Expected: Build succeeds with no errors.

**Step 4: Commit**

```bash
git add package.json bun.lock components/ui/
git commit -m "feat: add zustand, dnd-kit, nanoid, and shadcn dialog/popover/checkbox"
```

---

### Task 2: Types & Constants

**Files:**
- Create: `lib/types.ts`
- Create: `lib/constants.ts`

**Step 1: Create type definitions**

Create `lib/types.ts`:
```typescript
export type ColumnId = 'backlog' | 'todo' | 'in-progress' | 'review' | 'done';
export type Priority = 'high' | 'medium' | 'low';
export type Theme = 'light' | 'dark';

export interface Subtask {
  id: string;
  title: string;
  completed: boolean;
}

export interface Task {
  id: string;
  title: string;
  description?: string;
  status: ColumnId;
  priority: Priority;
  tagIds: string[];
  dueDate?: string;
  subtasks: Subtask[];
  order: number;
  createdAt: string;
  updatedAt: string;
}

export interface Tag {
  id: string;
  name: string;
  color: string;
}

export interface Column {
  id: ColumnId;
  title: string;
  taskIds: string[];
}

export interface Filters {
  search: string;
  tagIds: string[];
  priority: Priority | null;
  dueDateRange: { from?: string; to?: string } | null;
}
```

**Step 2: Create constants**

Create `lib/constants.ts`:
```typescript
import type { Column, ColumnId } from './types';

export const COLUMN_ORDER: ColumnId[] = [
  'backlog', 'todo', 'in-progress', 'review', 'done'
];

export const DEFAULT_COLUMNS: Record<ColumnId, Column> = {
  backlog: { id: 'backlog', title: 'Backlog', taskIds: [] },
  todo: { id: 'todo', title: 'Todo', taskIds: [] },
  'in-progress': { id: 'in-progress', title: 'In Progress', taskIds: [] },
  review: { id: 'review', title: 'Review', taskIds: [] },
  done: { id: 'done', title: 'Done', taskIds: [] },
};

export const PRIORITY_CONFIG = {
  high: { label: 'High', color: 'text-red-500', bg: 'bg-red-500/10' },
  medium: { label: 'Medium', color: 'text-yellow-500', bg: 'bg-yellow-500/10' },
  low: { label: 'Low', color: 'text-blue-500', bg: 'bg-blue-500/10' },
} as const;

export const DEFAULT_FILTERS = {
  search: '',
  tagIds: [],
  priority: null,
  dueDateRange: null,
} as const;
```

**Step 3: Verify types compile**

Run: `bunx tsc --noEmit`
Expected: No errors.

**Step 4: Commit**

```bash
git add lib/types.ts lib/constants.ts
git commit -m "feat: add type definitions and constants for todo kanban"
```

---

### Task 3: Zustand Store

**Files:**
- Create: `lib/store.ts`

**Step 1: Create the zustand store**

Create `lib/store.ts`:
```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';
import { nanoid } from 'nanoid';
import type { Task, Tag, Column, ColumnId, Priority, Filters, Theme, Subtask } from './types';
import { DEFAULT_COLUMNS, DEFAULT_FILTERS } from './constants';

interface TodoState {
  tasks: Record<string, Task>;
  columns: Record<ColumnId, Column>;
  tags: Record<string, Tag>;
  theme: Theme;
  filters: Filters;

  // Task CRUD
  addTask: (title: string, columnId: ColumnId) => string;
  updateTask: (id: string, updates: Partial<Omit<Task, 'id' | 'createdAt'>>) => void;
  deleteTask: (id: string) => void;
  moveTask: (taskId: string, fromCol: ColumnId, toCol: ColumnId, newIndex: number) => void;
  reorderTask: (columnId: ColumnId, fromIndex: number, toIndex: number) => void;

  // Subtask
  addSubtask: (taskId: string, title: string) => void;
  toggleSubtask: (taskId: string, subtaskId: string) => void;
  deleteSubtask: (taskId: string, subtaskId: string) => void;

  // Tags
  createTag: (name: string, color: string) => string;
  updateTag: (id: string, updates: Partial<Omit<Tag, 'id'>>) => void;
  deleteTag: (id: string) => void;

  // Filters
  setSearchQuery: (query: string) => void;
  setFilterTags: (tagIds: string[]) => void;
  setFilterPriority: (priority: Priority | null) => void;
  clearFilters: () => void;

  // Theme
  setTheme: (theme: Theme) => void;
  toggleTheme: () => void;
}

export const useTodoStore = create<TodoState>()(
  persist(
    (set, get) => ({
      tasks: {},
      columns: structuredClone(DEFAULT_COLUMNS),
      tags: {},
      theme: 'light',
      filters: { ...DEFAULT_FILTERS },

      addTask: (title, columnId) => {
        const id = nanoid();
        const now = new Date().toISOString();
        const task: Task = {
          id,
          title,
          status: columnId,
          priority: 'medium',
          tagIds: [],
          subtasks: [],
          order: get().columns[columnId].taskIds.length,
          createdAt: now,
          updatedAt: now,
        };
        set((state) => ({
          tasks: { ...state.tasks, [id]: task },
          columns: {
            ...state.columns,
            [columnId]: {
              ...state.columns[columnId],
              taskIds: [...state.columns[columnId].taskIds, id],
            },
          },
        }));
        return id;
      },

      updateTask: (id, updates) => {
        set((state) => {
          const existing = state.tasks[id];
          if (!existing) return state;
          return {
            tasks: {
              ...state.tasks,
              [id]: { ...existing, ...updates, updatedAt: new Date().toISOString() },
            },
          };
        });
      },

      deleteTask: (id) => {
        set((state) => {
          const task = state.tasks[id];
          if (!task) return state;
          const { [id]: _, ...remainingTasks } = state.tasks;
          return {
            tasks: remainingTasks,
            columns: {
              ...state.columns,
              [task.status]: {
                ...state.columns[task.status],
                taskIds: state.columns[task.status].taskIds.filter((tid) => tid !== id),
              },
            },
          };
        });
      },

      moveTask: (taskId, fromCol, toCol, newIndex) => {
        set((state) => {
          const task = state.tasks[taskId];
          if (!task) return state;
          const fromTaskIds = state.columns[fromCol].taskIds.filter((id) => id !== taskId);
          const toTaskIds = fromCol === toCol
            ? fromTaskIds
            : [...state.columns[toCol].taskIds];
          toTaskIds.splice(newIndex, 0, taskId);
          return {
            tasks: {
              ...state.tasks,
              [taskId]: { ...task, status: toCol, updatedAt: new Date().toISOString() },
            },
            columns: {
              ...state.columns,
              [fromCol]: { ...state.columns[fromCol], taskIds: fromTaskIds },
              ...(fromCol !== toCol
                ? { [toCol]: { ...state.columns[toCol], taskIds: toTaskIds } }
                : { [toCol]: { ...state.columns[toCol], taskIds: toTaskIds } }),
            },
          };
        });
      },

      reorderTask: (columnId, fromIndex, toIndex) => {
        set((state) => {
          const taskIds = [...state.columns[columnId].taskIds];
          const [moved] = taskIds.splice(fromIndex, 1);
          taskIds.splice(toIndex, 0, moved);
          return {
            columns: {
              ...state.columns,
              [columnId]: { ...state.columns[columnId], taskIds },
            },
          };
        });
      },

      addSubtask: (taskId, title) => {
        set((state) => {
          const task = state.tasks[taskId];
          if (!task) return state;
          const subtask: Subtask = { id: nanoid(), title, completed: false };
          return {
            tasks: {
              ...state.tasks,
              [taskId]: {
                ...task,
                subtasks: [...task.subtasks, subtask],
                updatedAt: new Date().toISOString(),
              },
            },
          };
        });
      },

      toggleSubtask: (taskId, subtaskId) => {
        set((state) => {
          const task = state.tasks[taskId];
          if (!task) return state;
          return {
            tasks: {
              ...state.tasks,
              [taskId]: {
                ...task,
                subtasks: task.subtasks.map((st) =>
                  st.id === subtaskId ? { ...st, completed: !st.completed } : st
                ),
                updatedAt: new Date().toISOString(),
              },
            },
          };
        });
      },

      deleteSubtask: (taskId, subtaskId) => {
        set((state) => {
          const task = state.tasks[taskId];
          if (!task) return state;
          return {
            tasks: {
              ...state.tasks,
              [taskId]: {
                ...task,
                subtasks: task.subtasks.filter((st) => st.id !== subtaskId),
                updatedAt: new Date().toISOString(),
              },
            },
          };
        });
      },

      createTag: (name, color) => {
        const id = nanoid();
        set((state) => ({
          tags: { ...state.tags, [id]: { id, name, color } },
        }));
        return id;
      },

      updateTag: (id, updates) => {
        set((state) => {
          const existing = state.tags[id];
          if (!existing) return state;
          return {
            tags: { ...state.tags, [id]: { ...existing, ...updates } },
          };
        });
      },

      deleteTag: (id) => {
        set((state) => {
          const { [id]: _, ...remainingTags } = state.tags;
          const updatedTasks = { ...state.tasks };
          for (const taskId of Object.keys(updatedTasks)) {
            const task = updatedTasks[taskId];
            if (task.tagIds.includes(id)) {
              updatedTasks[taskId] = {
                ...task,
                tagIds: task.tagIds.filter((tid) => tid !== id),
              };
            }
          }
          return { tags: remainingTags, tasks: updatedTasks };
        });
      },

      setSearchQuery: (search) => {
        set((state) => ({ filters: { ...state.filters, search } }));
      },

      setFilterTags: (tagIds) => {
        set((state) => ({ filters: { ...state.filters, tagIds } }));
      },

      setFilterPriority: (priority) => {
        set((state) => ({ filters: { ...state.filters, priority } }));
      },

      clearFilters: () => {
        set({ filters: { ...DEFAULT_FILTERS } });
      },

      setTheme: (theme) => set({ theme }),

      toggleTheme: () => {
        set((state) => ({ theme: state.theme === 'light' ? 'dark' : 'light' }));
      },
    }),
    {
      name: 'todo-app-storage',
      partialize: (state) => ({
        tasks: state.tasks,
        columns: state.columns,
        tags: state.tags,
        theme: state.theme,
      }),
    }
  )
);
```

**Step 2: Create selector helpers**

Add at the bottom of `lib/store.ts`:
```typescript
export function useFilteredTasksByColumn(columnId: ColumnId): Task[] {
  return useTodoStore((state) => {
    const column = state.columns[columnId];
    const { search, tagIds, priority, dueDateRange } = state.filters;

    return column.taskIds
      .map((id) => state.tasks[id])
      .filter((task): task is Task => {
        if (!task) return false;
        if (search) {
          const q = search.toLowerCase();
          if (!task.title.toLowerCase().includes(q) &&
              !(task.description?.toLowerCase().includes(q))) {
            return false;
          }
        }
        if (tagIds.length > 0 && !tagIds.some((id) => task.tagIds.includes(id))) {
          return false;
        }
        if (priority && task.priority !== priority) {
          return false;
        }
        if (dueDateRange) {
          if (!task.dueDate) return false;
          if (dueDateRange.from && task.dueDate < dueDateRange.from) return false;
          if (dueDateRange.to && task.dueDate > dueDateRange.to) return false;
        }
        return true;
      });
  });
}
```

**Step 3: Verify types compile**

Run: `bunx tsc --noEmit`
Expected: No errors.

**Step 4: Commit**

```bash
git add lib/store.ts
git commit -m "feat: add zustand store with task CRUD, tags, subtasks, filters, and persistence"
```

---

### Task 4: Theme Provider & Layout Update

**Files:**
- Create: `components/theme-provider.tsx`
- Modify: `app/layout.tsx`
- Modify: `app/page.tsx`

**Step 1: Create theme provider component**

Create `components/theme-provider.tsx`:
```typescript
"use client";

import { useEffect } from "react";
import { useTodoStore } from "@/lib/store";

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const theme = useTodoStore((s) => s.theme);

  useEffect(() => {
    const root = document.documentElement;
    if (theme === "dark") {
      root.classList.add("dark");
    } else {
      root.classList.remove("dark");
    }
  }, [theme]);

  // Detect system preference on first load (only if no persisted theme)
  useEffect(() => {
    const stored = localStorage.getItem("todo-app-storage");
    if (!stored) {
      const prefersDark = window.matchMedia("(prefers-color-scheme: dark)").matches;
      if (prefersDark) {
        useTodoStore.getState().setTheme("dark");
      }
    }
  }, []);

  return <>{children}</>;
}
```

**Step 2: Update layout.tsx**

Modify `app/layout.tsx`:
- Change metadata title to "Todo Kanban"
- Change description to "A Kanban-style task management app"
- Add `suppressHydrationWarning` to `<html>` tag (for dark class toggling)
- Wrap `{children}` with `<ThemeProvider>`

```typescript
import type { Metadata } from "next";
import { Geist, Geist_Mono, Inter } from "next/font/google";
import { ThemeProvider } from "@/components/theme-provider";
import "./globals.css";

const inter = Inter({ subsets: ["latin"], variable: "--font-sans" });

const geistSans = Geist({
  variable: "--font-geist-sans",
  subsets: ["latin"],
});

const geistMono = Geist_Mono({
  variable: "--font-geist-mono",
  subsets: ["latin"],
});

export const metadata: Metadata = {
  title: "Todo Kanban",
  description: "A Kanban-style task management app",
};

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <html lang="en" className={inter.variable} suppressHydrationWarning>
      <body
        className={`${geistSans.variable} ${geistMono.variable} antialiased`}
      >
        <ThemeProvider>{children}</ThemeProvider>
      </body>
    </html>
  );
}
```

**Step 3: Replace page.tsx with placeholder**

Replace `app/page.tsx`:
```typescript
import { BoardLayout } from "@/components/board-layout";

export default function Page() {
  return <BoardLayout />;
}
```

**Step 4: Create empty BoardLayout placeholder**

Create `components/board-layout.tsx`:
```typescript
"use client";

export function BoardLayout() {
  return (
    <div className="flex h-screen flex-col">
      <div className="p-4 text-center text-muted-foreground">
        Kanban board coming soon...
      </div>
    </div>
  );
}
```

**Step 5: Verify build**

Run: `bun run build`
Expected: Build succeeds.

**Step 6: Commit**

```bash
git add components/theme-provider.tsx components/board-layout.tsx app/layout.tsx app/page.tsx
git commit -m "feat: add theme provider, update layout, and scaffold board page"
```

---

### Task 5: Header Component

**Files:**
- Create: `components/header.tsx`
- Modify: `components/board-layout.tsx`

**Step 1: Create header component**

Create `components/header.tsx`:
```typescript
"use client";

import { Sun, Moon, Search } from "lucide-react";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { useTodoStore } from "@/lib/store";

export function Header() {
  const theme = useTodoStore((s) => s.theme);
  const toggleTheme = useTodoStore((s) => s.toggleTheme);
  const search = useTodoStore((s) => s.filters.search);
  const setSearchQuery = useTodoStore((s) => s.setSearchQuery);

  return (
    <header className="flex items-center justify-between border-b px-4 py-3">
      <h1 className="text-lg font-semibold">Todo Kanban</h1>

      <div className="flex items-center gap-2">
        <div className="relative">
          <Search className="absolute left-2.5 top-1/2 size-4 -translate-y-1/2 text-muted-foreground" />
          <Input
            placeholder="Search tasks..."
            value={search}
            onChange={(e) => setSearchQuery(e.target.value)}
            className="w-64 pl-8"
          />
        </div>

        <Button
          variant="ghost"
          size="icon"
          onClick={toggleTheme}
          aria-label="Toggle theme"
        >
          {theme === "light" ? <Moon className="size-4" /> : <Sun className="size-4" />}
        </Button>
      </div>
    </header>
  );
}
```

**Step 2: Wire header into board layout**

Update `components/board-layout.tsx`:
```typescript
"use client";

import { Header } from "@/components/header";

export function BoardLayout() {
  return (
    <div className="flex h-screen flex-col">
      <Header />
      <div className="flex-1 p-4 text-center text-muted-foreground">
        Kanban board coming soon...
      </div>
    </div>
  );
}
```

**Step 3: Verify dev server renders**

Run: `bun dev`
Expected: Page shows "Todo Kanban" header with search bar and theme toggle. Toggle switches light/dark mode.

**Step 4: Commit**

```bash
git add components/header.tsx components/board-layout.tsx
git commit -m "feat: add header with search bar and dark mode toggle"
```

---

### Task 6: TaskCard Component

**Files:**
- Create: `components/task-card.tsx`
- Create: `lib/date-utils.ts`

**Step 1: Create date utility helpers**

Create `lib/date-utils.ts`:
```typescript
export function getDueDateStatus(dueDate: string): 'overdue' | 'soon' | 'normal' {
  const today = new Date();
  today.setHours(0, 0, 0, 0);
  const due = new Date(dueDate);
  due.setHours(0, 0, 0, 0);

  const diffMs = due.getTime() - today.getTime();
  const diffDays = Math.ceil(diffMs / (1000 * 60 * 60 * 24));

  if (diffDays < 0) return 'overdue';
  if (diffDays <= 1) return 'soon';
  return 'normal';
}

export function formatDueDate(dueDate: string): string {
  const date = new Date(dueDate);
  return date.toLocaleDateString('ko-KR', {
    month: 'short',
    day: 'numeric',
  });
}
```

**Step 2: Create TaskCard component**

Create `components/task-card.tsx`:
```typescript
"use client";

import { forwardRef } from "react";
import { Calendar, GripVertical } from "lucide-react";
import { Card } from "@/components/ui/card";
import { Badge } from "@/components/ui/badge";
import { useTodoStore } from "@/lib/store";
import { PRIORITY_CONFIG } from "@/lib/constants";
import { getDueDateStatus, formatDueDate } from "@/lib/date-utils";
import { cn } from "@/lib/utils";
import type { Task } from "@/lib/types";

interface TaskCardProps {
  task: Task;
  onClick?: () => void;
  isDragging?: boolean;
  dragHandleProps?: React.HTMLAttributes<HTMLButtonElement>;
}

export const TaskCard = forwardRef<HTMLDivElement, TaskCardProps>(
  function TaskCard({ task, onClick, isDragging, dragHandleProps, ...props }, ref) {
    const tags = useTodoStore((s) => s.tags);
    const priorityCfg = PRIORITY_CONFIG[task.priority];
    const completedSubtasks = task.subtasks.filter((st) => st.completed).length;
    const totalSubtasks = task.subtasks.length;

    const dueDateStatus = task.dueDate ? getDueDateStatus(task.dueDate) : null;

    return (
      <Card
        ref={ref}
        size="sm"
        className={cn(
          "cursor-pointer transition-shadow hover:ring-2 hover:ring-ring/30",
          isDragging && "opacity-50 ring-2 ring-primary"
        )}
        onClick={onClick}
        {...props}
      >
        <div className="flex items-start gap-2 px-3 py-2">
          <button
            className="mt-0.5 shrink-0 cursor-grab text-muted-foreground hover:text-foreground active:cursor-grabbing"
            {...dragHandleProps}
            onClick={(e) => e.stopPropagation()}
          >
            <GripVertical className="size-4" />
          </button>

          <div className="min-w-0 flex-1 space-y-1.5">
            <p className="text-sm font-medium leading-snug">{task.title}</p>

            <div className="flex flex-wrap items-center gap-1">
              <Badge
                variant="secondary"
                className={cn("text-[10px]", priorityCfg.color, priorityCfg.bg)}
              >
                {priorityCfg.label}
              </Badge>

              {task.tagIds.map((tagId) => {
                const tag = tags[tagId];
                if (!tag) return null;
                return (
                  <Badge
                    key={tagId}
                    variant="outline"
                    className="text-[10px]"
                    style={{ borderColor: tag.color, color: tag.color }}
                  >
                    {tag.name}
                  </Badge>
                );
              })}
            </div>

            <div className="flex items-center gap-3 text-xs text-muted-foreground">
              {task.dueDate && (
                <span
                  className={cn(
                    "flex items-center gap-1",
                    dueDateStatus === "overdue" && "text-red-500",
                    dueDateStatus === "soon" && "text-orange-500"
                  )}
                >
                  <Calendar className="size-3" />
                  {formatDueDate(task.dueDate)}
                </span>
              )}

              {totalSubtasks > 0 && (
                <span className="flex items-center gap-1">
                  {completedSubtasks}/{totalSubtasks}
                  <div className="h-1 w-10 overflow-hidden rounded-full bg-muted">
                    <div
                      className="h-full rounded-full bg-primary transition-all"
                      style={{
                        width: `${(completedSubtasks / totalSubtasks) * 100}%`,
                      }}
                    />
                  </div>
                </span>
              )}
            </div>
          </div>
        </div>
      </Card>
    );
  }
);
```

**Step 3: Verify types compile**

Run: `bunx tsc --noEmit`
Expected: No errors.

**Step 4: Commit**

```bash
git add components/task-card.tsx lib/date-utils.ts
git commit -m "feat: add TaskCard component with priority badges, tags, due dates, subtask progress"
```

---

### Task 7: KanbanColumn Component

**Files:**
- Create: `components/kanban-column.tsx`
- Create: `components/add-task-inline.tsx`

**Step 1: Create inline task adder**

Create `components/add-task-inline.tsx`:
```typescript
"use client";

import { useState, useRef, useEffect } from "react";
import { Plus } from "lucide-react";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { useTodoStore } from "@/lib/store";
import type { ColumnId } from "@/lib/types";

interface AddTaskInlineProps {
  columnId: ColumnId;
}

export function AddTaskInline({ columnId }: AddTaskInlineProps) {
  const [isAdding, setIsAdding] = useState(false);
  const [title, setTitle] = useState("");
  const inputRef = useRef<HTMLInputElement>(null);
  const addTask = useTodoStore((s) => s.addTask);

  useEffect(() => {
    if (isAdding) inputRef.current?.focus();
  }, [isAdding]);

  function handleSubmit() {
    const trimmed = title.trim();
    if (trimmed) {
      addTask(trimmed, columnId);
      setTitle("");
    }
    setIsAdding(false);
  }

  if (!isAdding) {
    return (
      <Button
        variant="ghost"
        size="sm"
        className="w-full justify-start text-muted-foreground"
        onClick={() => setIsAdding(true)}
      >
        <Plus className="size-4" />
        Add task
      </Button>
    );
  }

  return (
    <Input
      ref={inputRef}
      placeholder="Task title..."
      value={title}
      onChange={(e) => setTitle(e.target.value)}
      onKeyDown={(e) => {
        if (e.key === "Enter") handleSubmit();
        if (e.key === "Escape") {
          setTitle("");
          setIsAdding(false);
        }
      }}
      onBlur={handleSubmit}
    />
  );
}
```

**Step 2: Create KanbanColumn component**

Create `components/kanban-column.tsx`:
```typescript
"use client";

import { useDroppable } from "@dnd-kit/core";
import {
  SortableContext,
  verticalListSortingStrategy,
} from "@dnd-kit/sortable";
import { useSortable } from "@dnd-kit/sortable";
import { CSS } from "@dnd-kit/utilities";
import { useFilteredTasksByColumn, useTodoStore } from "@/lib/store";
import { TaskCard } from "@/components/task-card";
import { AddTaskInline } from "@/components/add-task-inline";
import type { ColumnId, Task } from "@/lib/types";

interface KanbanColumnProps {
  columnId: ColumnId;
  title: string;
  onTaskClick: (task: Task) => void;
}

function SortableTaskCard({
  task,
  onTaskClick,
}: {
  task: Task;
  onTaskClick: (task: Task) => void;
}) {
  const {
    attributes,
    listeners,
    setNodeRef,
    transform,
    transition,
    isDragging,
  } = useSortable({ id: task.id });

  const style = {
    transform: CSS.Transform.toString(transform),
    transition,
  };

  return (
    <div ref={setNodeRef} style={style} {...attributes}>
      <TaskCard
        task={task}
        onClick={() => onTaskClick(task)}
        isDragging={isDragging}
        dragHandleProps={listeners}
      />
    </div>
  );
}

export function KanbanColumn({ columnId, title, onTaskClick }: KanbanColumnProps) {
  const tasks = useFilteredTasksByColumn(columnId);
  const totalCount = useTodoStore((s) => s.columns[columnId].taskIds.length);
  const { setNodeRef } = useDroppable({ id: columnId });

  return (
    <div className="flex w-72 shrink-0 flex-col rounded-lg bg-muted/50 p-2">
      <div className="mb-2 flex items-center justify-between px-2">
        <h2 className="text-sm font-semibold">{title}</h2>
        <span className="text-xs text-muted-foreground">{totalCount}</span>
      </div>

      <div
        ref={setNodeRef}
        className="flex min-h-[100px] flex-1 flex-col gap-2"
      >
        <SortableContext
          items={tasks.map((t) => t.id)}
          strategy={verticalListSortingStrategy}
        >
          {tasks.map((task) => (
            <SortableTaskCard
              key={task.id}
              task={task}
              onTaskClick={onTaskClick}
            />
          ))}
        </SortableContext>
      </div>

      <div className="mt-2">
        <AddTaskInline columnId={columnId} />
      </div>
    </div>
  );
}
```

**Step 3: Verify types compile**

Run: `bunx tsc --noEmit`
Expected: No errors.

**Step 4: Commit**

```bash
git add components/kanban-column.tsx components/add-task-inline.tsx
git commit -m "feat: add KanbanColumn with sortable task cards and inline task creation"
```

---

### Task 8: KanbanBoard with DnD

**Files:**
- Create: `components/kanban-board.tsx`
- Modify: `components/board-layout.tsx`

**Step 1: Create KanbanBoard component**

Create `components/kanban-board.tsx`:
```typescript
"use client";

import { useState, useCallback } from "react";
import {
  DndContext,
  DragOverlay,
  closestCorners,
  PointerSensor,
  TouchSensor,
  KeyboardSensor,
  useSensor,
  useSensors,
  type DragStartEvent,
  type DragEndEvent,
  type DragOverEvent,
} from "@dnd-kit/core";
import { sortableKeyboardCoordinates } from "@dnd-kit/sortable";
import { useTodoStore } from "@/lib/store";
import { COLUMN_ORDER } from "@/lib/constants";
import { KanbanColumn } from "@/components/kanban-column";
import { TaskCard } from "@/components/task-card";
import type { ColumnId, Task } from "@/lib/types";

interface KanbanBoardProps {
  onTaskClick: (task: Task) => void;
}

export function KanbanBoard({ onTaskClick }: KanbanBoardProps) {
  const columns = useTodoStore((s) => s.columns);
  const tasks = useTodoStore((s) => s.tasks);
  const moveTask = useTodoStore((s) => s.moveTask);
  const [activeTask, setActiveTask] = useState<Task | null>(null);

  const sensors = useSensors(
    useSensor(PointerSensor, { activationConstraint: { distance: 5 } }),
    useSensor(TouchSensor, {
      activationConstraint: { delay: 250, tolerance: 5 },
    }),
    useSensor(KeyboardSensor, {
      coordinateGetter: sortableKeyboardCoordinates,
    })
  );

  const findColumnByTaskId = useCallback(
    (taskId: string): ColumnId | null => {
      for (const colId of COLUMN_ORDER) {
        if (columns[colId].taskIds.includes(taskId)) {
          return colId;
        }
      }
      return null;
    },
    [columns]
  );

  function handleDragStart(event: DragStartEvent) {
    const task = tasks[event.active.id as string];
    if (task) setActiveTask(task);
  }

  function handleDragOver(event: DragOverEvent) {
    const { active, over } = event;
    if (!over) return;

    const activeId = active.id as string;
    const overId = over.id as string;

    const activeCol = findColumnByTaskId(activeId);
    // over could be a column id or a task id
    const overCol = COLUMN_ORDER.includes(overId as ColumnId)
      ? (overId as ColumnId)
      : findColumnByTaskId(overId);

    if (!activeCol || !overCol || activeCol === overCol) return;

    // Move task to new column at end
    const newIndex = columns[overCol].taskIds.length;
    moveTask(activeId, activeCol, overCol, newIndex);
  }

  function handleDragEnd(event: DragEndEvent) {
    const { active, over } = event;
    setActiveTask(null);

    if (!over) return;

    const activeId = active.id as string;
    const overId = over.id as string;

    const activeCol = findColumnByTaskId(activeId);
    if (!activeCol) return;

    // If dropped on same column, reorder
    const overCol = COLUMN_ORDER.includes(overId as ColumnId)
      ? (overId as ColumnId)
      : findColumnByTaskId(overId);

    if (!overCol) return;

    if (activeCol === overCol && activeId !== overId) {
      const taskIds = columns[activeCol].taskIds;
      const fromIndex = taskIds.indexOf(activeId);
      const toIndex = COLUMN_ORDER.includes(overId as ColumnId)
        ? taskIds.length - 1
        : taskIds.indexOf(overId);

      if (fromIndex !== toIndex && fromIndex !== -1 && toIndex !== -1) {
        useTodoStore.getState().reorderTask(activeCol, fromIndex, toIndex);
      }
    }
  }

  return (
    <DndContext
      sensors={sensors}
      collisionDetection={closestCorners}
      onDragStart={handleDragStart}
      onDragOver={handleDragOver}
      onDragEnd={handleDragEnd}
    >
      <div className="flex gap-4 overflow-x-auto p-4">
        {COLUMN_ORDER.map((colId) => (
          <KanbanColumn
            key={colId}
            columnId={colId}
            title={columns[colId].title}
            onTaskClick={onTaskClick}
          />
        ))}
      </div>

      <DragOverlay>
        {activeTask ? <TaskCard task={activeTask} /> : null}
      </DragOverlay>
    </DndContext>
  );
}
```

**Step 2: Wire KanbanBoard into BoardLayout**

Update `components/board-layout.tsx`:
```typescript
"use client";

import { useState } from "react";
import { Header } from "@/components/header";
import { KanbanBoard } from "@/components/kanban-board";
import type { Task } from "@/lib/types";

export function BoardLayout() {
  const [selectedTask, setSelectedTask] = useState<Task | null>(null);

  return (
    <div className="flex h-screen flex-col">
      <Header />
      <main className="flex-1 overflow-hidden">
        <KanbanBoard onTaskClick={setSelectedTask} />
      </main>
      {/* TaskDetailDialog will be added in Task 9 */}
    </div>
  );
}
```

**Step 3: Verify dev server**

Run: `bun dev`
Expected: Kanban board renders with 5 columns. Can add tasks inline. Can drag tasks between columns.

**Step 4: Commit**

```bash
git add components/kanban-board.tsx components/board-layout.tsx
git commit -m "feat: add KanbanBoard with drag-and-drop between columns"
```

---

### Task 9: TaskDetailDialog

**Files:**
- Create: `components/task-detail-dialog.tsx`
- Modify: `components/board-layout.tsx`

**Step 1: Create TaskDetailDialog**

Create `components/task-detail-dialog.tsx`:
```typescript
"use client";

import { useState, useEffect } from "react";
import { Trash2, Plus } from "lucide-react";
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
} from "@/components/ui/dialog";
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
  AlertDialogTrigger,
} from "@/components/ui/alert-dialog";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import { Label } from "@/components/ui/label";
import { Badge } from "@/components/ui/badge";
import { Checkbox } from "@/components/ui/checkbox";
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/select";
import { useTodoStore } from "@/lib/store";
import { COLUMN_ORDER, PRIORITY_CONFIG } from "@/lib/constants";
import type { Task, ColumnId, Priority } from "@/lib/types";

interface TaskDetailDialogProps {
  task: Task | null;
  open: boolean;
  onOpenChange: (open: boolean) => void;
}

export function TaskDetailDialog({
  task,
  open,
  onOpenChange,
}: TaskDetailDialogProps) {
  const updateTask = useTodoStore((s) => s.updateTask);
  const deleteTask = useTodoStore((s) => s.deleteTask);
  const moveTask = useTodoStore((s) => s.moveTask);
  const addSubtask = useTodoStore((s) => s.addSubtask);
  const toggleSubtask = useTodoStore((s) => s.toggleSubtask);
  const deleteSubtask = useTodoStore((s) => s.deleteSubtask);
  const tags = useTodoStore((s) => s.tags);
  const columns = useTodoStore((s) => s.columns);

  const [title, setTitle] = useState("");
  const [description, setDescription] = useState("");
  const [newSubtaskTitle, setNewSubtaskTitle] = useState("");

  // Sync local state when task changes
  useEffect(() => {
    if (task) {
      setTitle(task.title);
      setDescription(task.description ?? "");
    }
  }, [task]);

  if (!task) return null;

  // Re-read task from store to get latest state (subtasks, etc.)
  const currentTask = useTodoStore.getState().tasks[task.id];
  if (!currentTask) return null;

  function handleTitleBlur() {
    const trimmed = title.trim();
    if (trimmed && trimmed !== currentTask.title) {
      updateTask(currentTask.id, { title: trimmed });
    }
  }

  function handleDescriptionBlur() {
    if (description !== (currentTask.description ?? "")) {
      updateTask(currentTask.id, { description: description || undefined });
    }
  }

  function handleStatusChange(newStatus: string) {
    const fromCol = currentTask.status;
    const toCol = newStatus as ColumnId;
    if (fromCol !== toCol) {
      const newIndex = columns[toCol].taskIds.length;
      moveTask(currentTask.id, fromCol, toCol, newIndex);
    }
  }

  function handlePriorityChange(newPriority: string) {
    updateTask(currentTask.id, { priority: newPriority as Priority });
  }

  function handleDueDateChange(e: React.ChangeEvent<HTMLInputElement>) {
    updateTask(currentTask.id, {
      dueDate: e.target.value || undefined,
    });
  }

  function handleTagToggle(tagId: string) {
    const newTagIds = currentTask.tagIds.includes(tagId)
      ? currentTask.tagIds.filter((id) => id !== tagId)
      : [...currentTask.tagIds, tagId];
    updateTask(currentTask.id, { tagIds: newTagIds });
  }

  function handleAddSubtask() {
    const trimmed = newSubtaskTitle.trim();
    if (trimmed) {
      addSubtask(currentTask.id, trimmed);
      setNewSubtaskTitle("");
    }
  }

  function handleDelete() {
    deleteTask(currentTask.id);
    onOpenChange(false);
  }

  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent className="max-h-[85vh] overflow-y-auto sm:max-w-lg">
        <DialogHeader>
          <DialogTitle className="sr-only">Edit Task</DialogTitle>
        </DialogHeader>

        <div className="space-y-4">
          {/* Title */}
          <Input
            value={title}
            onChange={(e) => setTitle(e.target.value)}
            onBlur={handleTitleBlur}
            className="text-lg font-semibold"
            onKeyDown={(e) => {
              if (e.key === "Enter") e.currentTarget.blur();
            }}
          />

          {/* Description */}
          <div className="space-y-1.5">
            <Label>Description</Label>
            <Textarea
              value={description}
              onChange={(e) => setDescription(e.target.value)}
              onBlur={handleDescriptionBlur}
              placeholder="Add a description..."
              rows={3}
            />
          </div>

          {/* Status & Priority */}
          <div className="grid grid-cols-2 gap-3">
            <div className="space-y-1.5">
              <Label>Status</Label>
              <Select
                value={currentTask.status}
                onValueChange={handleStatusChange}
              >
                <SelectTrigger>
                  <SelectValue />
                </SelectTrigger>
                <SelectContent>
                  {COLUMN_ORDER.map((colId) => (
                    <SelectItem key={colId} value={colId}>
                      {columns[colId]?.title ?? colId}
                    </SelectItem>
                  ))}
                </SelectContent>
              </Select>
            </div>

            <div className="space-y-1.5">
              <Label>Priority</Label>
              <Select
                value={currentTask.priority}
                onValueChange={handlePriorityChange}
              >
                <SelectTrigger>
                  <SelectValue />
                </SelectTrigger>
                <SelectContent>
                  {(["high", "medium", "low"] as Priority[]).map((p) => (
                    <SelectItem key={p} value={p}>
                      {PRIORITY_CONFIG[p].label}
                    </SelectItem>
                  ))}
                </SelectContent>
              </Select>
            </div>
          </div>

          {/* Due Date */}
          <div className="space-y-1.5">
            <Label>Due Date</Label>
            <Input
              type="date"
              value={currentTask.dueDate ?? ""}
              onChange={handleDueDateChange}
            />
          </div>

          {/* Tags */}
          <div className="space-y-1.5">
            <Label>Tags</Label>
            <div className="flex flex-wrap gap-1.5">
              {Object.values(tags).map((tag) => (
                <Badge
                  key={tag.id}
                  variant={
                    currentTask.tagIds.includes(tag.id) ? "default" : "outline"
                  }
                  className="cursor-pointer"
                  style={
                    currentTask.tagIds.includes(tag.id)
                      ? { backgroundColor: tag.color, borderColor: tag.color }
                      : { borderColor: tag.color, color: tag.color }
                  }
                  onClick={() => handleTagToggle(tag.id)}
                >
                  {tag.name}
                </Badge>
              ))}
              {Object.keys(tags).length === 0 && (
                <span className="text-xs text-muted-foreground">
                  No tags yet. Create tags from the filter bar.
                </span>
              )}
            </div>
          </div>

          {/* Subtasks */}
          <div className="space-y-1.5">
            <Label>
              Subtasks
              {currentTask.subtasks.length > 0 &&
                ` (${currentTask.subtasks.filter((s) => s.completed).length}/${currentTask.subtasks.length})`}
            </Label>
            <div className="space-y-1">
              {currentTask.subtasks.map((st) => (
                <div key={st.id} className="flex items-center gap-2">
                  <Checkbox
                    checked={st.completed}
                    onCheckedChange={() => toggleSubtask(currentTask.id, st.id)}
                  />
                  <span
                    className={
                      st.completed
                        ? "flex-1 text-sm text-muted-foreground line-through"
                        : "flex-1 text-sm"
                    }
                  >
                    {st.title}
                  </span>
                  <Button
                    variant="ghost"
                    size="icon-xs"
                    onClick={() => deleteSubtask(currentTask.id, st.id)}
                  >
                    <Trash2 className="size-3" />
                  </Button>
                </div>
              ))}
            </div>
            <div className="flex gap-2">
              <Input
                placeholder="Add subtask..."
                value={newSubtaskTitle}
                onChange={(e) => setNewSubtaskTitle(e.target.value)}
                onKeyDown={(e) => {
                  if (e.key === "Enter") handleAddSubtask();
                }}
                className="flex-1"
              />
              <Button variant="ghost" size="icon" onClick={handleAddSubtask}>
                <Plus className="size-4" />
              </Button>
            </div>
          </div>

          {/* Delete */}
          <div className="border-t pt-4">
            <AlertDialog>
              <AlertDialogTrigger asChild>
                <Button variant="destructive" size="sm">
                  <Trash2 className="size-4" />
                  Delete Task
                </Button>
              </AlertDialogTrigger>
              <AlertDialogContent>
                <AlertDialogHeader>
                  <AlertDialogTitle>Delete task?</AlertDialogTitle>
                  <AlertDialogDescription>
                    This will permanently delete &quot;{currentTask.title}&quot; and all its subtasks.
                  </AlertDialogDescription>
                </AlertDialogHeader>
                <AlertDialogFooter>
                  <AlertDialogCancel>Cancel</AlertDialogCancel>
                  <AlertDialogAction onClick={handleDelete}>
                    Delete
                  </AlertDialogAction>
                </AlertDialogFooter>
              </AlertDialogContent>
            </AlertDialog>
          </div>
        </div>
      </DialogContent>
    </Dialog>
  );
}
```

**Step 2: Wire dialog into BoardLayout**

Update `components/board-layout.tsx`:
```typescript
"use client";

import { useState } from "react";
import { Header } from "@/components/header";
import { KanbanBoard } from "@/components/kanban-board";
import { TaskDetailDialog } from "@/components/task-detail-dialog";
import type { Task } from "@/lib/types";

export function BoardLayout() {
  const [selectedTask, setSelectedTask] = useState<Task | null>(null);

  return (
    <div className="flex h-screen flex-col">
      <Header />
      <main className="flex-1 overflow-hidden">
        <KanbanBoard onTaskClick={setSelectedTask} />
      </main>
      <TaskDetailDialog
        task={selectedTask}
        open={selectedTask !== null}
        onOpenChange={(open) => {
          if (!open) setSelectedTask(null);
        }}
      />
    </div>
  );
}
```

**Step 3: Verify dev server**

Run: `bun dev`
Expected: Click on a task card opens modal. Can edit title, description, status, priority, due date, subtasks. Delete confirmation works.

**Step 4: Commit**

```bash
git add components/task-detail-dialog.tsx components/board-layout.tsx
git commit -m "feat: add task detail dialog with full CRUD, subtasks, and delete confirmation"
```

---

### Task 10: FilterBar Component

**Files:**
- Create: `components/filter-bar.tsx`
- Modify: `components/board-layout.tsx`

**Step 1: Create FilterBar component**

Create `components/filter-bar.tsx`:
```typescript
"use client";

import { X } from "lucide-react";
import { Button } from "@/components/ui/button";
import { Badge } from "@/components/ui/badge";
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/select";
import { useTodoStore } from "@/lib/store";
import { PRIORITY_CONFIG } from "@/lib/constants";
import type { Priority } from "@/lib/types";

export function FilterBar() {
  const tags = useTodoStore((s) => s.tags);
  const filters = useTodoStore((s) => s.filters);
  const setFilterTags = useTodoStore((s) => s.setFilterTags);
  const setFilterPriority = useTodoStore((s) => s.setFilterPriority);
  const clearFilters = useTodoStore((s) => s.clearFilters);

  const hasActiveFilters =
    filters.tagIds.length > 0 || filters.priority !== null;

  function handleTagToggle(tagId: string) {
    const newTagIds = filters.tagIds.includes(tagId)
      ? filters.tagIds.filter((id) => id !== tagId)
      : [...filters.tagIds, tagId];
    setFilterTags(newTagIds);
  }

  if (Object.keys(tags).length === 0 && !hasActiveFilters) return null;

  return (
    <div className="flex items-center gap-2 border-b px-4 py-2">
      <span className="text-xs font-medium text-muted-foreground">Filter:</span>

      {/* Tag filters */}
      {Object.values(tags).map((tag) => (
        <Badge
          key={tag.id}
          variant={filters.tagIds.includes(tag.id) ? "default" : "outline"}
          className="cursor-pointer text-[10px]"
          style={
            filters.tagIds.includes(tag.id)
              ? { backgroundColor: tag.color, borderColor: tag.color }
              : { borderColor: tag.color, color: tag.color }
          }
          onClick={() => handleTagToggle(tag.id)}
        >
          {tag.name}
        </Badge>
      ))}

      {/* Priority filter */}
      <Select
        value={filters.priority ?? "all"}
        onValueChange={(v) =>
          setFilterPriority(v === "all" ? null : (v as Priority))
        }
      >
        <SelectTrigger className="h-6 w-28 text-xs">
          <SelectValue placeholder="Priority" />
        </SelectTrigger>
        <SelectContent>
          <SelectItem value="all">All priorities</SelectItem>
          {(["high", "medium", "low"] as Priority[]).map((p) => (
            <SelectItem key={p} value={p}>
              {PRIORITY_CONFIG[p].label}
            </SelectItem>
          ))}
        </SelectContent>
      </Select>

      {/* Clear filters */}
      {hasActiveFilters && (
        <Button
          variant="ghost"
          size="xs"
          onClick={clearFilters}
          className="text-muted-foreground"
        >
          <X className="size-3" />
          Clear
        </Button>
      )}
    </div>
  );
}
```

**Step 2: Wire FilterBar into BoardLayout**

Update `components/board-layout.tsx` — add `<FilterBar />` between `<Header />` and `<main>`:
```typescript
"use client";

import { useState } from "react";
import { Header } from "@/components/header";
import { FilterBar } from "@/components/filter-bar";
import { KanbanBoard } from "@/components/kanban-board";
import { TaskDetailDialog } from "@/components/task-detail-dialog";
import type { Task } from "@/lib/types";

export function BoardLayout() {
  const [selectedTask, setSelectedTask] = useState<Task | null>(null);

  return (
    <div className="flex h-screen flex-col">
      <Header />
      <FilterBar />
      <main className="flex-1 overflow-hidden">
        <KanbanBoard onTaskClick={setSelectedTask} />
      </main>
      <TaskDetailDialog
        task={selectedTask}
        open={selectedTask !== null}
        onOpenChange={(open) => {
          if (!open) setSelectedTask(null);
        }}
      />
    </div>
  );
}
```

**Step 3: Verify dev server**

Run: `bun dev`
Expected: Filter bar appears when tags exist. Can filter by tag/priority. Clear button resets all filters.

**Step 4: Commit**

```bash
git add components/filter-bar.tsx components/board-layout.tsx
git commit -m "feat: add filter bar with tag and priority filtering"
```

---

### Task 11: TagManageDialog

**Files:**
- Create: `components/tag-manage-dialog.tsx`
- Modify: `components/header.tsx`

**Step 1: Create TagManageDialog component**

Create `components/tag-manage-dialog.tsx`:
```typescript
"use client";

import { useState } from "react";
import { Trash2, Plus } from "lucide-react";
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
} from "@/components/ui/dialog";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Badge } from "@/components/ui/badge";
import { useTodoStore } from "@/lib/store";

const PRESET_COLORS = [
  "#ef4444", "#f97316", "#eab308", "#22c55e",
  "#3b82f6", "#8b5cf6", "#ec4899", "#6b7280",
];

export function TagManageDialog({ children }: { children: React.ReactNode }) {
  const tags = useTodoStore((s) => s.tags);
  const createTag = useTodoStore((s) => s.createTag);
  const deleteTag = useTodoStore((s) => s.deleteTag);

  const [name, setName] = useState("");
  const [color, setColor] = useState(PRESET_COLORS[0]);

  function handleCreate() {
    const trimmed = name.trim();
    if (trimmed) {
      createTag(trimmed, color);
      setName("");
      setColor(PRESET_COLORS[0]);
    }
  }

  return (
    <Dialog>
      <DialogTrigger asChild>{children}</DialogTrigger>
      <DialogContent className="sm:max-w-sm">
        <DialogHeader>
          <DialogTitle>Manage Tags</DialogTitle>
        </DialogHeader>

        <div className="space-y-4">
          {/* Existing tags */}
          <div className="space-y-2">
            {Object.values(tags).map((tag) => (
              <div key={tag.id} className="flex items-center justify-between">
                <Badge
                  variant="outline"
                  style={{ borderColor: tag.color, color: tag.color }}
                >
                  {tag.name}
                </Badge>
                <Button
                  variant="ghost"
                  size="icon-xs"
                  onClick={() => deleteTag(tag.id)}
                >
                  <Trash2 className="size-3" />
                </Button>
              </div>
            ))}
            {Object.keys(tags).length === 0 && (
              <p className="text-sm text-muted-foreground">No tags yet.</p>
            )}
          </div>

          {/* Create tag */}
          <div className="space-y-2 border-t pt-3">
            <Label>New Tag</Label>
            <div className="flex gap-2">
              <Input
                placeholder="Tag name"
                value={name}
                onChange={(e) => setName(e.target.value)}
                onKeyDown={(e) => {
                  if (e.key === "Enter") handleCreate();
                }}
                className="flex-1"
              />
              <Button variant="ghost" size="icon" onClick={handleCreate}>
                <Plus className="size-4" />
              </Button>
            </div>

            <div className="flex gap-1.5">
              {PRESET_COLORS.map((c) => (
                <button
                  key={c}
                  className="size-6 rounded-full ring-offset-background transition-all focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring"
                  style={{
                    backgroundColor: c,
                    outline: color === c ? "2px solid currentColor" : "none",
                    outlineOffset: "2px",
                  }}
                  onClick={() => setColor(c)}
                />
              ))}
            </div>
          </div>
        </div>
      </DialogContent>
    </Dialog>
  );
}
```

**Step 2: Add tag management button to Header**

Update `components/header.tsx` — add a Tags button next to the theme toggle:
```typescript
"use client";

import { Sun, Moon, Search, Tags } from "lucide-react";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { useTodoStore } from "@/lib/store";
import { TagManageDialog } from "@/components/tag-manage-dialog";

export function Header() {
  const theme = useTodoStore((s) => s.theme);
  const toggleTheme = useTodoStore((s) => s.toggleTheme);
  const search = useTodoStore((s) => s.filters.search);
  const setSearchQuery = useTodoStore((s) => s.setSearchQuery);

  return (
    <header className="flex items-center justify-between border-b px-4 py-3">
      <h1 className="text-lg font-semibold">Todo Kanban</h1>

      <div className="flex items-center gap-2">
        <div className="relative">
          <Search className="absolute left-2.5 top-1/2 size-4 -translate-y-1/2 text-muted-foreground" />
          <Input
            placeholder="Search tasks..."
            value={search}
            onChange={(e) => setSearchQuery(e.target.value)}
            className="w-64 pl-8"
          />
        </div>

        <TagManageDialog>
          <Button variant="ghost" size="icon" aria-label="Manage tags">
            <Tags className="size-4" />
          </Button>
        </TagManageDialog>

        <Button
          variant="ghost"
          size="icon"
          onClick={toggleTheme}
          aria-label="Toggle theme"
        >
          {theme === "light" ? <Moon className="size-4" /> : <Sun className="size-4" />}
        </Button>
      </div>
    </header>
  );
}
```

**Step 3: Verify dev server**

Run: `bun dev`
Expected: Tags icon in header opens manage dialog. Can create tags with color. Tags appear in filter bar and task detail dialog.

**Step 4: Commit**

```bash
git add components/tag-manage-dialog.tsx components/header.tsx
git commit -m "feat: add tag management dialog with color picker"
```

---

### Task 12: Clean Up & Final Build

**Files:**
- Delete: `components/component-example.tsx`
- Delete: `components/example.tsx`

**Step 1: Remove unused example components**

Run:
```bash
rm components/component-example.tsx components/example.tsx
```

**Step 2: Run lint**

Run: `bun run lint`
Expected: No errors (fix any that appear).

**Step 3: Run production build**

Run: `bun run build`
Expected: Build succeeds with no errors.

**Step 4: Commit**

```bash
git add -A
git commit -m "chore: remove example components and verify clean build"
```

---

### Task 13: Manual Smoke Test

**No files changed — verification only.**

Start `bun dev` and verify these flows:

1. **Task CRUD:** Create task in Todo column, click to edit title/description, delete with confirmation
2. **Drag & Drop:** Drag task from Todo to In Progress, reorder within column
3. **Tags:** Create 2+ tags with different colors, assign to tasks, filter by tag
4. **Priority:** Set High/Medium/Low on tasks, filter by priority
5. **Due Date:** Set due dates, verify overdue/soon colors
6. **Subtasks:** Add subtasks, toggle completion, check progress bar on card
7. **Search:** Type in search bar, verify cards filter by title/description
8. **Dark Mode:** Toggle theme, verify all components render correctly
9. **Persistence:** Refresh page, verify all data persists
10. **Combined Filters:** Apply tag + priority filter together, clear all

Fix any bugs found during testing and commit fixes.
