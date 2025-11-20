---
title: "Anatomy of Hexagonal Architecture - Part 2"
date: 2025-07-03
draft: false
summary: "Understanding Entities and Domain layer"
images:
    - "/images/blog/hexagonal-architecture.png"
slug: "anatomy-of-hexagonal-architecture-2"
---

As mentioned in the previous blog post, we'll now start with the actual implementation.

Just as the construction of a home starts with its foundation, we're going to begin by designing the base of our application - the Domain/Core layer by defining entities for our app.

To understand the entities of our app, we first need to understand the features of our application and what it does.

Todoist has many capabilities, but for learning purposes, we'll focus on just a few core features:
- A **User** owns many **Projects**
- A **Project** has many **Tasks**
- A **User** can create many **Tasks** under a **Project**

From these requirements, we can define the entities of our app:
- User
- Project
- Task

Let's define the properties of these entities:

```
User -
    id -> UUID                       // Will act as userId
    name -> string                   // name of user
    email -> string                  // email of user
    createdAt -> Date                // user creation date-time
    updatedAt -> Date                // user updation date-time

Project -
    id -> UUID                       // Will act as projectId          
    title -> string                  // title of project
    color -> string                  // color associated with that project, will act as identifier
    createdBy -> UUID                // Creator of project
    createdAt -> Date                // project creation date-time
    updatedAt -> Date                // project updation date-time

Task -
    id -> UUID                       // Will act as taskId
    title -> string                  // title of task
    description -> string            // description of task
    projectId -> UUID                // projectId under which this task is created
    createdBy -> UUID                // Owner/Creator of task
    dueDate -> Date?                 // Optional property, due date by which this task should be completed
    priority -> 1 | 2 | 3 | 4        // Priority of the task, lower number = higher priority
    isCompleted -> boolean           // is given task completed or not
    createdAt -> Date                // task creation date-time
    updatedAt -> Date                // task updation date-time
```

Great! Our entity definition is complete. Now we need to translate this logic into readable code.

Before we start coding, let's prepare our arsenal - tools, project initialization, folder structure, etc.

## Tools
- <a class="Reference-link" href="https://bun.sh/">Bun</a> - will be our runtime
- <a class="Reference-link" href="https://supabase.com/">Supabase</a> - will be our database
- <a class="Reference-link" href="https://orm.drizzle.team/">Drizzle</a> - will be our ORM (Object Relational Mapping) tool
- <a class="Reference-link" href="https://zod.dev/">Zod</a> - will be our validator

## Project Setup

Run the following commands to initialize the project and create the initial folder structure:

```bash
# Create a blank project with bun having name mytodoist
bun init mytodoist
cd mytodoist

# Install initial packages
bun add drizzle-orm postgres dotenv zod
bun add -D drizzle-kit tsx

# create required files/folders
/mytodoist> touch .env
/mytodoist> mkdir adapters cmd core infrastructure migrations
/mytodoist> mkdir core/domain
/mytodoist/core/domain> cd core/domain
/mytodoist/core/domain>  touch users.domain.ts projects.domain.ts tasks.domain.ts
```

## Domain Entity Implementation

Now we'll define our domain entities:

```ts {linenos=inline}
// users.domain.ts

import { pgTable, varchar, uuid, timestamp } from "drizzle-orm/pg-core";
import z from "zod";

// Domain Model

export const UserModelSchema = z.object({
  id: z.string().uuid(),
  name: z
    .string()
    .min(1, "Name is required")
    .max(255, "Name must be less than 255 characters"),
  email: z
    .string()
    .email("Invalid email format")
    .max(255, "Email must be less than 255 characters"),
  createdAt: z.date().optional(),
  updatedAt: z.date().optional(),
});

export type UserModel = z.infer<typeof UserModelSchema>;

// Database Table Definition

export const UsersTable = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  name: varchar("name", { length: 255 }).notNull(),
  email: varchar("email", { length: 255 }).notNull().unique(),
  createdAt: timestamp("created_at").defaultNow().notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).$onUpdate(
    () => new Date()
  ),
});
```

```ts {linenos=inline}
// projects.domain.ts

import { z } from "zod";
import { pgTable, uuid, varchar, timestamp } from "drizzle-orm/pg-core";
import { UsersTable } from "./users.domain";

// Domain Models schema

export const ProjectModelSchema = z.object({
  id: z.string().uuid(),
  title: z.string().min(1).max(255),
  createdBy: z.string().uuid(),
  createdAt: z.date().optional(),
  updatedAt: z.date().optional(),
});

export type ProjectModel = z.infer<typeof ProjectModelSchema>;

// Database Table Definition - Drizzle ORM

export const ProjectsTable = pgTable("projects", {
  id: uuid("id").primaryKey().defaultRandom(),
  title: varchar("title", { length: 255 }).notNull(),
  createdBy: uuid("created_by")
    .references(() => UsersTable.id)
    .notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).$onUpdate(
    () => new Date()
  ),
});
```

```ts {linenos=inline}
// tasks.domain.ts

import { z } from "zod";
import { pgTable, uuid, varchar, timestamp, pgEnum, boolean } from "drizzle-orm/pg-core";
import { UsersTable } from "./users.domain";
import { ProjectsTable } from "./projects.domain";

// Domain Models schema

const TasksPriority = ["1", "2", "3", "4"] as const;

export const TaskModelSchema = z.object({
  id: z.string().uuid(),
  title: z
    .string()
    .min(1, "Title is required")
    .max(255, "Title must be less than 255 characters"),
  description: z
    .string()
    .min(1, "Description is required")
    .max(255, "Description must be less than 255 characters")
    .optional(),
  projectId: z.string().uuid(),
  createdBy: z.string().uuid(),
  dueDate: z.date().optional(),
  priority: z.enum(TasksPriority),
  isCompleted: z.boolean(),
  createdAt: z.date().optional(),
  updatedAt: z.date().optional(),
});

export type TaskModel = z.infer<typeof TaskModelSchema>;

// Database Table Definition - Drizzle ORM

export const TasksPriorityEnum = pgEnum("tasks_priority", ["1", "2", "3", "4"]);

export const TasksTable = pgTable("tasks", {
  id: uuid("id").primaryKey().defaultRandom(),
  title: varchar("title", { length: 255 }).notNull(),
  description: varchar("description", { length: 255 }),
  projectId: uuid("project_id")
    .references(() => ProjectsTable.id)
    .notNull(),
  createdBy: uuid("created_by")
    .references(() => UsersTable.id)
    .notNull(),
  dueDate: timestamp("due_date", { withTimezone: true }),
  priority: TasksPriorityEnum("priority").notNull(),
  isCompleted: boolean("is_completed").default(false).notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).$onUpdate(
    () => new Date()
  ),
});
```

## Configuration Setup

Now, follow these steps to complete the setup:

1. **Environment Configuration**: Add your database URL to the `.env` file (this is the secure way to handle sensitive data):
    ```
    // .env

    DATABASE_URL=<SUPABASE_URL>
    ```

2. **Package.json Scripts**: Add essential commands to your `package.json`:
    ```json
    // package.json

    {
        "scripts": {
            "start": "bun run index.ts",
            "db:generate": "drizzle-kit generate",
            "db:migrate": "drizzle-kit migrate"
        }
    }
    ```

3. **Drizzle Configuration**: Create a `drizzle.config.ts` file in the root folder and add the following code:
    ```ts {linenos=inline}
    // drizzle.config.ts

    import 'dotenv/config';
    import { defineConfig } from 'drizzle-kit';

    export default defineConfig({
        // Drizzle will pick up DB schemas from here
        // and sync with Supabase
        schema: './core/domain/*.domain.ts',

        // All future incremental DB changes will be stored here
        // representing the current state/schema of tables
        out: './migrations',

        // Database type
        dialect: 'postgresql',
        dbCredentials: {
            // Database connection string
            url: process.env.DATABASE_URL as string,
        },
    });
    ```

4. **Database Migration**: Run these two commands in your terminal:
    ```bash
    bun run db:generate
    bun run db:migrate
    ```

Voila! You should now see the tables created in your Supabase project under Database → Tables.

## Understanding Our Domain Layer

This completes our Domain layer implementation. Before ending this blog post, let me also explain our folder structure.

As mentioned in the previous blog, the domain is part of our core. We've placed it in the `core` folder, and there are other folders we created that will make more sense as we progress further in this series:

```
├── adapters
├── cmd
├── core                        // Core of our application
│   └── domain                  // Domain layer
│       ├── projects.domain.ts
│       ├── tasks.domain.ts
│       └── users.domain.ts
├── drizzle.config.ts
├── index.ts
├── infrastructure
├── migrations
├── package.json
```

## What's Coming Next

In the next part of this series, we'll be moving outside of the core and implementing ports, services, and repositories.

Stay tuned for hands on coding where theory meets reality!