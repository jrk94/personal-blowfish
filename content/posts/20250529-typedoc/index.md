---
title: "Instant Customization Documentation"
description: "Using Type Doc for instant documentation"
summary: "Using Type Doc for instant documentation"
categories: ["IoT"]
tags: ["Documentation"]
##externalUrl: ""
date: 2025-05-29
draft: false
authors:
  - Roque
---

Documentation is often dealt with as an afterthought of software development. It is very hard to convince developers to write documentation. But developers do write code and do like their code to be readable and maintanable. There are tools that use that skill of writing good code and transform it into documentation. Bring forth the often laughed concept of code being the best documentation.

Connect IoT customization is built with typescript and [typedoc](https://typedoc.org/) is the premier tool for typescript documentation generation.

## What is TypeDoc?

At its core, **TypeDoc is a documentation generator for TypeScript projects**. It takes your TypeScript source code, leverages its rich type information, and parses your JSDoc-style (or [TSDoc](https://tsdoc.org/)-compliant) comments to create beautiful, browsable HTML API documentation. Think of it as your project's personal archivist, meticulously cataloging every class, interface, function, and variable.

## Why TypeDoc?

1. **TypeScript-Native Understanding**: Unlike generic documentation tools, TypeDoc is built from the ground up for TypeScript. It inherently understands the nuances of TypeScript's type system – interfaces, generics, union types, intersection types, access modifiers, and more. This means it accurately represents your API's structure without requiring verbose, redundant comments.

2. **Leverages Your Existing Comment**s: TypeDoc embraces the well-established JSDoc syntax (and the more modern TSDoc specification). If you're already documenting your functions and classes with JSDoc comments, TypeDoc will readily consume them, enriching the generated output. This means less new syntax to learn and more leverage from your existing work.

3. **Rich, Navigable Output**: The generated documentation is typically in a clean, HTML format, complete with:

- Search functionality: Quickly find what you're looking for.
- Hierarchical navigation: Easily explore modules, classes, and members.
- Syntax highlighting: Code examples are readable and clear.
- Cross-referencing: Links to related types and members within your documentation.

4. **Customization**: TypeDoc offers various themes and extensive configuration options, allowing you to tailor the look and feel of your documentation to match your project's branding or integrate seamlessly into a larger documentation website.

---

## How TypeDoc Works (The Simplified Flow)

1. **Installation**: You install TypeDoc as a development dependency in your project.

2. **Configuration**: You tell TypeDoc where your source files are (src/index.ts for example) and where you want the output (--out docs).

3. **Comment Parsing**: TypeDoc scans your TypeScript files for declarations (classes, functions, etc.) and their associated JSDoc/TSDoc comments.

4. **Type Information Integration**: It analyzes the TypeScript types (e.g., string, MyInterface, Array<T>) and incorporates this crucial information into the documentation.

5. **HTML Generation**: TypeDoc renders all this information into a set of interlinked HTML files (and supporting CSS/JS), typically with an index.html as the entry point.

## Getting Started with TypeDoc

Let's look at a quick example to see how simple it is:

1. Install TypeDoc:

```Bash
npm install typedoc --save-dev
```

2. Add JSDoc/TSDoc Comments to Your Code:

```TypeScript
/**
 *
 * This task transforms CFX Units Processed data into a set of IoT Post Events,
 * specifically for oven telemetry. It processes readings and setpoints from
 * oven zones and dispatches them to a telemetry system.
 *
 *
 * This task is activated by its `activate` input. Upon activation, it iterates
 * through provided oven data, constructs telemetry payloads, and posts them
 * using `SystemCalls.postTelemetry`. It handles success and error conditions,
 * emitting appropriate output signals.
 *
 * It's designed to work with structured oven data, matching readings with
 * corresponding setpoints and enriching the data with relevant tags and ISA95
 * hierarchy information.
 *
 * ### Inputs
 * * `activate`: `any` - Triggers the execution of the task. Set to any value to activate.
 * * `instance`: `System.LBOS.Cmf.Foundation.BusinessObjects.Entity` - The system entity (e.g., a specific oven) to which the telemetry belongs.
 * * `material`: `string` - The material currently being processed in the oven.
 * * `values`: `OvenData[]` - An array of oven data, where each `OvenData` object contains readings and setpoints for a specific zone.
 * * `valuesTimestamp`: `Date` - The timestamp to associate with all processed telemetry values.
 *
 * ### Outputs
 * * `success`: `boolean` - Emits `true` when all telemetry events are successfully posted.
 * * `error`: `Error` - Emits an `Error` object if the task fails to post telemetry for any reason.
 *
 * ### Settings
 * See {@link PostOvenTelemetrySettings} for configurable properties like application name,
 * retry mechanisms, and more.
 */
@Task.Task()
export class PostOvenTelemetryTask extends TaskBase implements PostOvenTelemetrySettings {

    /**
     * Called when one or more input values have changed.
     *
     * @param changes - An object representing the changed inputs.
     * @remarks
     * This method checks for activation of the task, processes oven telemetry data, 
     * and sends it to the system via `SystemCalls.postTelemetry`. It emits
     * success or error signals based on the result.
     */
    public override async onChanges(changes: Task.Changes): Promise<void> { ...

      
    /**
     * Initializes the task.
     *
     * @remarks
     * Registers any required handlers and ensures the settings are sanitized
     * against default values before execution begins.
     */

    public override async onInit(): Promise<void> { ...
```

3. Add a Script to package.json:

```JSON
{
  "name": "@criticalmanufacturing/connect-iot-controller-engine-custom-dataplatform-tasks",
  "version": "1.0.0",
  ...
  "scripts": {
    ...
    "doc:build": "npx typedoc --entryPointStrategy Expand src",
    "doc:start": "open-cli docs/index.html",
    "doc:build:start": "npm run doc:build && npm run doc:start"
  },
  "devDependencies": {
    "typedoc": "^0.25.0",
    "open-cli": "^8.0.0" // optional
  }
}
```

4. Generate Your Docs:

```Bash
npm run doc:build
```

After running this command, you'll find a docs/ folder in your project root, containing the generated HTML documentation. Open docs/index.html in your browser, you can also run, if you install `open-cli`:

```Bash
npm run doc:start
```

## Using it with Connect IoT

Connect IoT scaffolding already creates methods and classes ready to use TypeDoc. In this example. I have a connect iot package called `@criticalmanufacturing/connect-iot-controller-engine-custom-dataplatform-tasks`, with one task and a set of utilities.

<video controls width="100%">
  <source src="https://image.j-roque.com/posts/20250529-typedoc/TypeDocSingleTask.mp4" type="video/mp4">
</video>

Notice how from the index we can see all the code tree and have an easy way to see the project. You can also use the search function to find any method.

## Final Thoughts

For any serious TypeScript project, TypeDoc is an invaluable tool. It streamlines the documentation process, leverages the power of TypeScript's type system, and produces high-quality, maintainable API reference documentation. By incorporating TypeDoc into your development workflow, you empower your fellow developers, simplify onboarding, and ultimately contribute to a more robust and understandable codebase. Give it a try – your future self (and your team) will thank you!