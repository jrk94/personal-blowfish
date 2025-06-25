---
title: "Part II - Sharing Injected Components in Connect IoT"
description: "How to create code that is reused throughout the lifecycle"
summary: "How to create code that is reused throughout the lifecycle"
categories: ["IoT"]
tags: ["SQLite", "Customization"]
##externalUrl: ""
date: 2025-06-20
draft: false
authors:
  - Roque
---

This blog post will be part of a series of blog posts where we show case an example of how we can use a third party dependency like `SQLite` in our Connect IoT code. For this second blog post the goal is to show how iot components are injected and how we can have components that are long lived throughout the controller lifecycle. This is important for our SQLite example as we want just one instance of sqlite to be instantiated and then reused by all tasks. The part II assumes you have read and understood [part I](https://j-roque.com/posts/20250617-sqlite-i/).

## Overview

In this blog post we will start by understanding how tasks and converters are injected and then we will follow up with how we can inject our own components.

For implementations like in the case of SQLite, we want to avoid having each task creating it's own class responsible for creating a connection the database and performing actions. What is ideal is to centralize the connection to the database and then reuse the same singleton to centralize all database actions. This makes working with the database much easier, efficient and predictable.

This is even more important as tasks are somewhat ephemeral in nature (not completely as they can instantiate things like settings and other actions in the onInit() hook). It is a good practice to keep tasks ephemeral and for them to not have long lived behaviors, like for example keeping a database connection open or storing information that leaves beyond one activation. A task is an atomic action that should not need to know what happened before or after, for storing contextual information we can use things like the persistency singleton or even our new SQLite implementation. The task will then delegate the context to a persistency layer and only concern itself with applying logic. 

For some use cases like sub-workflows, standalone workflows or control flow tasks will be completely ephemeral and will only be created when they are activated or when the workflow plan is calculated.

Therefore it is important to have a way to share objects across the whole controller context. In Connect IoT we support the injection of classes in the context of a task, a workflow plan or across the whole controller.

Notice that this is not new, when you are handling things like the logger, the datastore (to handle the persistency layer) or the system, what you are actually doing is manipulating containers that were already made available in the dependency injection container by the default implementation. What we are going to show here is how we can create our own custom components to inject in the context.

## Entrypoint

When you perform the `cmf new iot task` command one of the elements of the scaffolding generated is the `index.ts` file. Each task or converter that is created with the scaffolding command `cmf new iot task` / `cmf new iot converter` will automatically add an entry into the index.ts file.

![Index TS](https://image.j-roque.com/posts/20250625-sqlite-ii/indexts.png)

When the controller process starts it will load all the tasks and converters declared in the index.ts file. It will look for the class with attribute `@Task.Task()` or `@Converter.Converter()`.

![Entrypoint](https://image.j-roque.com/posts/20250625-sqlite-ii/entrypoint.png)

In the controller start cycle it will import all the custom packages and their components.

![Controller Start Cycle](https://image.j-roque.com/posts/20250625-sqlite-ii/controllerstartcycle_darkmode.png)

## Creating Shareable Code

In our SQLite implementation we will want to have a common implementation that is then available to all our tasks for our SQLite CRUD.

### SQLite Manager

Let's create a new file `sqlite/sqliteManager.ts`. This class `SQLiteManager` will be an injectable in our dependency container. For handling, the dependency injection we are using [inversifyjs](https://github.com/inversify/InversifyJS), but we also provide easy attributes you can use.

In our class we can also access all the components that were injected in our container of dependency injection. This is very useful as we can have access to our logging system and other helpful components.

![SQLite Manager](https://image.j-roque.com/posts/20250625-sqlite-ii/sqlitemanager.png)

Looking at our class:

```ts
import Database from "better-sqlite3"; // SQLite Package
import * as path from "path";
import * as fs from "node:fs";
import {
    Dependencies,
    DI,
    System,
    TYPES
} from "@criticalmanufacturing/connect-iot-controller-engine";

@DI.Injectable()
export class SQLiteManager {

    private _db;
    private _dbLocation: string;
    private _tableSchemas: Set<string>;

    @DI.Inject(TYPES.Dependencies.Logger)
    private _logger: Dependencies.Logger;

    @DI.Inject(TYPES.System.PersistedDataStore)
    private _dataStore: System.DataStore;

    public startSQLLite(dbPath = "iot.db", cleanupTimerInterval = 60000) {
        // Retrieve persistency location from the configurations
        const persistencyLocation = path.join(this._dataStore["_handler"]["_config"]["path"], "SQLite", this._dataStore["_controllerId"].replace("/", "_"));

        fs.mkdirSync(persistencyLocation, { recursive: true });
        this._dbLocation = path.join(persistencyLocation, dbPath);

        // Create SQLite database file
        this._db = new Database(this._dbLocation);
         // Track created tables
        this._tableSchemas = new Set();
        this._logger.debug(`Started SQLite DB at ${this._dbLocation}`);

        // Start ttl timer
        this.scheduleCleanup(cleanupTimerInterval);
    }
...
```

In our class we can see that a lot of things are happening, this is because we already have some code relevant to our SQLite implementation. For now, let's focus on how we flag our class as able to be added to our dependency injection container, by using the `@DI.Injectable()` attribute. Also, we can see that we can access other classes injected in our dependency injection container by using `@DI.Inject` and then the name of our container. By using these attributes we have marked our class as injectable.

### Task Injection

Now we need to add a provider for this class. We can do so by modifying a bit our tasks to be able to act as providers of this component.

In the `StoreSQLiteTask` we will now add an inject for our SQLiteManager and a provider to instantiate it.

We will need to create new entrypoint for our task, that acts as a provider for this new component.

```ts
@Task.TaskModule({
    task: StoreSQLiteTask, // Our Task
    providers: [
        {
            class: SQLiteManager, // Component that we are injecting
            isSingleton: true, // Should this component be a Singleton for the whole Controller
            symbol: "GlobalSQLiteManagerHandler", // Name that will be used to Inject the container in the dependency injection
            scope: Task.ProviderScope.Controller, // Injection scope (Local; WorkflowPlan; Controller)
        }
    ]
})
export class StoreSQLiteModule { }
```

Here is where we can control the behavior of our component. We will say what class are we injecting and what is it name in the dependency injection container, if there is only one instance or if there can be multiple and finally if it's going to be shared across all task activations (Local), across a workflow plan execution (WorkflowPlan) or to all tasks in the Controller.

In the `index.ts` file we will now modify it to have as an entrypoint, not the default `Task.Task()` but the task module.

![Module Entrypoint](https://image.j-roque.com/posts/20250625-sqlite-ii/moduleentrypoint.png)

We can now inject and use our SQLiteManager component.

```ts
    /**
     * This is the representation of the SQLite manager
     */
    @DI.Inject("GlobalSQLiteManagerHandler")
    private _sqliteManager: SQLiteManager;
```

The name of the inject must match the one given in the symbol field of the module providers.

![Shared Injectable](https://image.j-roque.com/posts/20250625-sqlite-ii/sharedinjectable.png)

Now our `GlobalSQLiteManagerHandler` is able to be used by any task and converter in any package in this controller.

## Showing it Working

Let's change our code of our SQLiteManager a bit so we can see that it is called.

```ts
import Database from "better-sqlite3"; // SQLite Package
import * as path from "path";
import * as fs from "node:fs";
import {
    Dependencies,
    DI,
    System,
    TYPES
} from "@criticalmanufacturing/connect-iot-controller-engine";

@DI.Injectable()
export class SQLiteManager {

    private _db;
    private _dbLocation: string;
    private _tableSchemas: Set<string>;

    @DI.Inject(TYPES.Dependencies.Logger)
    private _logger: Dependencies.Logger;

    @DI.Inject(TYPES.System.PersistedDataStore)
    private _dataStore: System.DataStore;

    public startSQLLite(dbPath = "iot.db", cleanupTimerInterval = 60000) {
        this._logger.warning(`Please implement the SQLiteManager`);
    }
...
```

We will then have a simple workflow with a timer and two sequential store SQLite tasks.

![Shared Instance](https://image.j-roque.com/posts/20250625-sqlite-ii/sharedinstance.gif)

What we are able to see is that our SQLiteManager at first is instantiated but without a `_db`. Only when the first run calls the `startSQLLite` method will it create a `_db`. In the second call we are able to see that the `_db` created in the first call is still the same and we can reuse it.

With this we can now create a set of methods in the class `SQLiteManager` that are available for any task that injects it. By using:

```ts
    /**
     * This is the representation of the SQLite manager
     */
    @DI.Inject("GlobalSQLiteManagerHandler")
    private _sqliteManager: SQLiteManager;
```

If for a given task you want to defer the inject to wait for example, for a setup task to start your class you can use the Optional attribute `@DI.Optional()`.

## Final Thoughts

With this we have solved how to share code between all our tasks and converters. This is very helpful, to create reusable classes and utilities shared across different contexts. In the next and final part we will see a full implementation and with all of this working.