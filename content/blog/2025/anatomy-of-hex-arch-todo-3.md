---
title: "Anatomy of Hexagonal Architecture - Part 3"
date: 2025-07-12
draft: false
summary: "Understanding Ports, Services, and Repositories"
images:
    - "/images/blog/hexagonal-architecture.png"
slug: "anatomy-of-hexagonal-architecture-3"
---

As mentioned in the previous blog post, we'll now start implementing Ports, Services, and Repositories.

These are simple and understandable concepts. Let me explain them in one line:

- **Ports** are interface definitions for Domain entities
- **Repositories** handle interactions with external systems like databases, caches, queues, and other APIs
- **Services** contain the business logic and call **Repositories**, which are implementations of **Ports**

Simple enough! Let's dive in and learn more through implementation.

> **Note**: To keep this blog post manageable, I'll implement only the User entity. As practice, you can implement the same for Projects and Tasks.

## Implementing Ports

Run the following commands to create the initial files:

```bash
/mytodoist> mkdir core/ports
/mytodoist> touch core/ports/users.ports.ts
```

Now, let's think about what actions we can perform on the User entity, along with their requirements and return values:

- **Create a user** - Needs user data - Returns created User
- **Update a user** - Needs user data - Returns updated User  
- **Delete a user** - Needs user ID - Returns nothing
- **Get user by ID** - Needs user ID - Returns User or NULL (if not found)
- **Get user by Email** - Needs user email - Returns User or NULL (if not found)
- **Get list of users** - No requirements - Returns list of users

Now, let's create the definitions for these actions:

```ts
// users.ports.ts

import type { UserModel } from "../domain/users.domain";

// Repository Port Interface
// This interface defines the methods that the User Repository will implement.
// It acts as a contract for the data access layer to interact with user data.

export interface UsersRepoPort {
  create(user: UserModel): Promise<UserModel | null>;
  getById(id: string): Promise<UserModel | null>;
  getByEmail(email: string): Promise<UserModel | null>;
  update(id: string, userData: Partial<UserModel>): Promise<UserModel | null>;
  delete(id: string): Promise<void>;
  list(): Promise<UserModel[] | []>;
}
```

## Implementing Repositories

Run the following commands to create the initial files:

```bash
/mytodoist> mkdir -p adapters/secondary/persistence/postgresql/users
/mytodoist> touch adapters/secondary/persistence/postgresql/users/index.ts
```

That's quite a few folders! Let me explain the folder structure:

- `adapters/secondary` - As mentioned in the first blog, **Repositories** are secondary adapters
- `adapters/secondary/persistence` - We are storing/persisting data in a database
- `adapters/secondary/persistence/postgresql` - The name of the database where we're storing data

Inside `adapters/secondary`, we can also have folders/files according to our requirements:
- `/queue` for pushing data to queues
- `/cache` for storing data in cache
- etc.

Below is the implementation of the users repository, which interacts with a PostgreSQL database. It takes a Drizzle client as a dependency to interact with the database and returns an object with various methods that follow the structure of **UsersRepoPort**.

```ts
// adapters/secondary/persistence/postgresql/users/index.ts

import type { PostgresJsDatabase } from "drizzle-orm/postgres-js";
import type { UsersRepoPort } from "../../../../../core/ports/users.ports";
import { UsersTable } from "../../../../../core/domain/users.domain";
import { eq } from "drizzle-orm";

const REPO_NAME = "Postgresql-UsersRepo";

export const usersRepo = ({
  drizzleClient,
}: {
  drizzleClient: PostgresJsDatabase;
}): UsersRepoPort => {
  return {
    create: async (user) => {
      try {
        const [result] = await drizzleClient
          .insert(UsersTable)
          .values(user)
          .returning({
            id: UsersTable.id,
            name: UsersTable.name,
            email: UsersTable.email,
          });
        return result || null;
      } catch (error: Error | any) {
        throw new Error(`Error in ${REPO_NAME}:create:${error.message}`);
      }
    },

    getById: async (id) => {
      try {
        const [result] = await drizzleClient
          .select({
            id: UsersTable.id,
            name: UsersTable.name,
            email: UsersTable.email,
          })
          .from(UsersTable)
          .where(eq(UsersTable.id, id));

        if (!result) {
          throw new Error("User not found with given ID");
        }

        return result;
      } catch (error: Error | any) {
        throw new Error(`Error in ${REPO_NAME}:getById:${error.message}`);
      }
    },

    getByEmail: async (email) => {
      try {
        const [result] = await drizzleClient
          .select({
            id: UsersTable.id,
            name: UsersTable.name,
            email: UsersTable.email,
          })
          .from(UsersTable)
          .where(eq(UsersTable.email, email));

        if (!result) {
          throw new Error("User not found with given email");
        }

        return result;
      } catch (error: Error | any) {
        throw new Error(`Error in ${REPO_NAME}:getByEmail:${error.message}`);
      }
    },

    update: async (id, userData) => {
      try {
        const [updatedUser] = await drizzleClient
          .update(UsersTable)
          .set(userData)
          .where(eq(UsersTable.id, id))
          .returning({
            id: UsersTable.id,
            name: UsersTable.name,
            email: UsersTable.email,
          });

        return updatedUser || null;
      } catch (error: Error | any) {
        throw new Error(`Error in ${REPO_NAME}:update:${error.message}`);
      }
    },

    delete: async (id) => {
      try {
        await drizzleClient
          .delete(UsersTable)
          .where(eq(UsersTable.id, id));
      } catch (error: Error | any) {
        throw new Error(`Error in ${REPO_NAME}:delete:${error.message}`);
      }
    },

    list: async () => {
      try {
        const results = await drizzleClient
          .select({
            id: UsersTable.id,
            name: UsersTable.name,
            email: UsersTable.email,
          })
          .from(UsersTable);

        return results;
      } catch (error: Error | any) {
        throw new Error(`Error in ${REPO_NAME}:list:${error.message}`);
      }
    },
  };
};
```

## Implementing Services

Run the following command to create the initial file:

```bash
/mytodoist> mkdir core/services
/mytodoist> touch core/services/users.service.ts 
```

Similar to **Repositories**, we need to implement an interface for **Services**. We'll add it to the same file where we defined the **Repository** interfaces:

```ts
// users.ports.ts

// ...existing code...

export interface UsersServicePort {
  registerUser(user: UserModel): Promise<UserModel | null>;
  getUserById(id: string): Promise<UserModel | null>;
  getUserByEmail(email: string): Promise<UserModel | null>;
  updateUserProfile(
    id: string,
    userData: Partial<UserModel>
  ): Promise<UserModel | null>;
  deleteUserAccount(id: string): Promise<void>;
  getUsers(): Promise<UserModel[] | []>;
}
```

Now let's implement the service based on the interface above:

```ts
// core/services/users.service.ts

import type { UsersRepoPort, UsersServicePort } from "../ports/users.ports";

export const usersService = ({
  usersRepo,
}: {
  usersRepo: UsersRepoPort;
}): UsersServicePort => {
  return {
    registerUser: async (user) => {
      try {
        const createdUser = await usersRepo.create(user);
        return createdUser;
      } catch (error) {
        console.log(error);
        throw new Error("User registration failed");
      }
    },
    getUserById: async (userId) => {
      try {
        const user = await usersRepo.getById(userId);
        return user;
      } catch (error) {
        console.log(error);
        throw new Error("Error fetching user");
      }
    },
    getUserByEmail: async (email) => {
      try {
        const user = await usersRepo.getByEmail(email);
        return user;
      } catch (error) {
        console.log(error);
        throw new Error("Error fetching user by email");
      }
    },
    updateUserProfile: async (userId, userData) => {
      try {
        const updatedUser = await usersRepo.update(userId, userData);
        return updatedUser;
      } catch (error) {
        console.log(error);
        throw new Error("Error updating user profile");
      }
    },
    deleteUserAccount: async (userId) => {
      try {
        await usersRepo.delete(userId);
      } catch (error) {
        console.log(error);
        throw new Error("Error deleting user account");
      }
    },
    getUsers: async () => {
      try {
        const users = await usersRepo.list();
        return users;
      } catch (error) {
        console.log(error);
        throw new Error("Error fetching users");
      }
    },
  };
};
```

## Understanding the Difference: Services vs Repositories

Services and repositories might look similar at first glance, but services contain business logic and can operate on multiple entities by accepting multiple repositories, while repositories operate on a single entity.

Let's take an example to understand this better. Suppose we want to:
1. Create a user
2. Store it in cache
3. Push it to a queue

Here's how we would implement this:

1. **Pass multiple repositories** to the users service: users repository, cache repository, queue repository
2. **Check email uniqueness** using the users repository:
   - If email exists → return error
   - If email doesn't exist → create user
3. **Cache user data** for frequent access using the cache repository
4. **Queue user data** asynchronously for analytics using the queue repository
5. **Return success**

Notice that our business logic is tool/tech agnostic. Even if we change our cache from Redis to another tool, or move from our inhouse queue implementation to AWS SQS or other third party technology, we just need to implement the new adapter and pass it to the users service. (In upcoming blogs, we'll see how to pass these dependencies to services.)

## Understanding Our Updated Structure

Our folder structure now looks like this:

```
├── adapters
│   └── secondary
│       └── persistence
│           └── postgresql
│               └── users
│                   └── index.ts
├── cmd
├── core                        // Core of our application
│   ├── domain                  // Domain layer
│   │   ├── projects.domain.ts
│   │   ├── tasks.domain.ts
│   │   └── users.domain.ts
│   ├── ports                   // Port definitions
│   │   └── users.ports.ts
│   └── services                // Business logic layer
│       └── users.service.ts
├── drizzle.config.ts
├── index.ts
├── infrastructure
├── migrations
├── package.json
```

That's enough for this part! Let's meet in the next blog of our series.

## What's Coming Next

In the next part of this series, we'll complete our application by implementing Adapters (especially Primary Adapters).

Stay tuned for hands on coding where theory meets reality!