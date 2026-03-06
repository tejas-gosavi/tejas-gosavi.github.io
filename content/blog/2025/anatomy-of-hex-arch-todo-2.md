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
    id -> integer                    // Will act as userId
    name -> string                   // name of user
    email -> string                  // email of user

Project -
    id -> integer                    // Will act as projectId          
    title -> string                  // title of project
    createdBy -> integer             // Creator of project

Task -
    id -> integer                    // Will act as taskId
    title -> string                  // title of task
    description -> string            // description of task
    projectId -> integer             // projectId under which this task is created
    createdBy -> integer             // Owner/Creator of task
    isCompleted -> boolean           // is given task completed or not
```

Great! Our entity definition is complete. Now we need to translate this logic into readable code.

Before we start coding, let's prepare our arsenal - tools, project initialization, folder structure, etc.

## Tools
- <a class="Reference-link" href="https://bun.sh/">Bun</a> - will be our runtime
- <a class="Reference-link" href="https://sqlite.org/">SQLite</a> - will be our database
- <a class="Reference-link" href="https://orm.drizzle.team/">Drizzle</a> - will be our ORM (Object Relational Mapping) tool
- <a class="Reference-link" href="https://zod.dev/">Zod</a> - will be our validator

## Project Setup

Run the following commands to initialize the project and create the initial folder structure:

```bash
# Create a blank project with bun having name mytodoist
bun init mytodoist
cd mytodoist

# Install initial packages
bun add drizzle-orm @libsql/client dotenv zod
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

import { sqliteTable, integer, text } from "drizzle-orm/sqlite-core";
import z from "zod";

// Domain Model

export const UserModelSchema = z.object({
  id: z.int(),
  name: z
    .string()
    .min(1, "Name is required")
    .max(255, "Name must be less than 255 characters"),
  email: z
    .string()
    .email("Invalid email format")
    .max(255, "Email must be less than 255 characters"),
});

export type UserModel = z.infer<typeof UserModelSchema>;

// Database Table Definition

export const UsersTable = sqliteTable("users", {
  id: integer("id").primaryKey(),
  name: text("name", { length: 255 }).notNull(),
  email: text("email", { length: 255 }).notNull().unique(),
});
```

```ts {linenos=inline}
// projects.domain.ts

import { z } from "zod";
import { sqliteTable, integer, text } from "drizzle-orm/sqlite-core";
import { UsersTable } from "./users.domain";

// Domain Models schema

export const ProjectModelSchema = z.object({
  id: z.int(),
  title: z
    .string()
    .min(1, "Project Title is required")
    .max(255, "Project Title must be less than 255 characters"),
  createdBy: z.int(),
});

export type ProjectModel = z.infer<typeof ProjectModelSchema>;

// Database Table Definition - Drizzle ORM

export const ProjectsTable = sqliteTable("projects", {
  id: integer("id").primaryKey(),
  title: text("title", { length: 255 }).notNull(),
  createdBy: integer("created_by")
    .references(() => UsersTable.id)
    .notNull(),
});
```

```ts {linenos=inline}
// tasks.domain.ts

import { z } from "zod";
import { sqliteTable, integer, text } from "drizzle-orm/sqlite-core";
import { UsersTable } from "./users.domain";
import { ProjectsTable } from "./projects.domain";
import { sql } from "drizzle-orm";

// Domain Models schema

export const TaskModelSchema = z.object({
  id: z.int(),
  name: z
    .string()
    .min(1, "Task Name is required")
    .max(255, "Task Name must be less than 255 characters"),
  description: z
    .string()
    .min(1, "Description is required")
    .max(255, "Description must be less than 255 characters")
    .optional(),
  projectId: z.int(),
  createdBy: z.int(),
  isCompleted: z.boolean(),
});

export type TaskModel = z.infer<typeof TaskModelSchema>;

// Database Table Definition - Drizzle ORM

export const TasksTable = sqliteTable("tasks", {
  id: integer("id").primaryKey(),
  name: text("name", { length: 255 }).notNull(),
  description: text("description", { length: 255 }),
  projectId: integer("project_id")
    .references(() => ProjectsTable.id)
    .notNull(),
  createdBy: integer("created_by")
    .references(() => UsersTable.id)
    .notNull(),
  isCompleted: integer("is_completed", { mode: "boolean" }).default(
    sql`(abs(0))`
  ),
});
```

## Configuration Setup

Now, follow these steps to complete the setup:

1. **Environment Configuration**: Add your database URL to the `.env` file (this is the secure way to handle sensitive data):
    ```
    // .env

    DB_FILE_NAME=file:local.db
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

    import "dotenv/config";
    import { defineConfig } from "drizzle-kit";

    export default defineConfig({
      // Drizzle will pick up DB schemas from here
      // and sync with Local SQLite DB
      schema: "./core/domain/*.domain.ts",

      // All future incremental DB changes will be stored here
      // representing the current state/schema of tables
      out: "./migrations",

      // Database type
      dialect: "sqlite",
      dbCredentials: {
        // Database connection string
        url: process.env.DB_FILE_NAME!,
      },
    });
    ```

4. **Database Migration**: Run these two commands in your terminal:
    ```bash
    bun run db:generate
    bun run db:migrate
    ```

Voila! Tables are created in your local SQLite database.

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