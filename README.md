# Electron Logging

<details>
<summary>Table of Contents</summary>

- [Intro](#intro)
- [electron-log](#electron-log)
- [Process Specific Logging](#process-specific-logging)
  - [Logging `MAIN PROCESS` & `RENDER PROCESS(es)`](#logging-main-process--render-processes)
  - [Logging `UTILITY PROCESS(es)`](#logging-utility-processes)
- [Structured Logging with `electron-log`](#structured-logging-with-electron-log)
- [Development vs Production Logging](#development-vs-production-logging)
- [Security Considerations](#security-considerations)
- [A Note on Alternative Logging Libraries](#a-note-on-alternative-logging-libraries)

> _[code example](https://github.com/MrT3313/ELECTRON-EXAMPLE)_

</details>
</br>

# Intro

<div align="center">
  <p>
    The <a href='https://github.com/Mr-T-Writing/Electron-101/blob/main/README.md'>unique multi-process architecture of Electron</a> comprising a <code>MAIN PROCESS</code> and one or more <code>RENDER PROCESS(es) / UTILITY PROCESS(es)</code> introduces complexities not found in traditional web or Node.js development. Without proper logging, tracking errors across these isolated processes quickly becomes a nightmare.
  </p>

  <p>To solve these pesky debugging problems I would like to introduce 🥁🥁🥁🥁</p>

  <h1 style="border-bottom: none; margin: 0; margin-top: 10px">
    🎉🎉 <a href="https://github.com/megahertz/electron-log">electron-log</a> 🎉🎉
  </h1>

  <p>
    which is a widely accepted ... <strong>logging...</strong> solution for ... <strong>Electron...</strong>
  </p>

  </br>

  <img src="./assets/hmm-suspect.gif">
</div>

# electron-log

`electron-log` established a unified logging system across all Electron processes, ensuring logs from the `MAIN PROCESS`, `RENDER PROCESS(es)`, and `UTILITY PROCESS(es)` are captured, organized, shown, and stored effectively.  

This logging approach is designed to:

1.  **Automatically Centralize Renderer Logs via IPC:** The `RENDER PROCESS(es)` logger (`electron-log/renderer`) automatically sends all logs to the `MAIN PROCESS` logger (`electron-log/main`) via `Inter-Process Communication (IPC)`. 
    
    This automation is triggered when `log.initialize()` is called in the `MAIN PROCESS`.

    ```typescript
    // In your main Electron process
    import log from 'electron-log/main';

    // Initialize electron-log for the main process.
    // This also sets up the IPC listener for logs from renderer processes.
    log.initialize(); 
    ```

2.  **Consistent Formatting:** A standard log message format is applied across all processes and log transports/outputs _(console and file)_.

    Transport formats are defined in the `electron-log/main` instance. Logs sent from `electron-log/renderer` will be formatted by the main process according to these definitions.

    - `log.transports.file.format = '{h}:{i}:{s} [{processType}{scope}] [{level}] > {text}'`
    - `log.transports.console.format = '{h}:{i}:{s} [{processType}{scope}] [{level}] > {text}'`

3.  **Structured Log Storage:** Logs are stored in a predictable directory structure.

    - **Windows**: `%APPDATA%\{APP_NAME}\logs\...`
    - **macOS**: `~/Library/Application Support/{APP_NAME}/logs/...`
    - **Linux**: `~/.config/{APP_NAME}/logs/...`

    You can configure a custom log path using the `log.transports.file.resolvePathFn(...)` function.

4.  **Independent Utility Logs:** `UTILITY PROCESS(es)` require their own separate `electron-log/main` instance to write logs directly.
    
    While we could configure `UTILITY PROCESS(es)` with different logging approaches (just as `RENDER PROCESS(es)` _could_ handle their own logging independently), maintaining consistency is key to a unified logging strategy.
    
    By using the same `log.transports.file.resolvePathFn(...)` and `log.transports.<file/console>.format` configurations across all processes, their separate logging logic integrate seamlessly into the application's overall logs.

## Log Rotation and Archiving with `electron-log`

You can control the maximum size of a log file using `log.transports.file.maxSize`. When a log file reaches this size, `electron-log` automatically renames it (`main.old.log`) and creates a fresh `main.log` file. Set it to `0` to disable rotation (default is 1MB).  

> `log.transports.file.maxSize = 5 * 1024 * 1024; // 5MB`

> [!IMPORTANT]
>
> A significant limitation to be aware of is that the `maxSize` check is primarily performed only when the logger is initialized. If a log file already exceeds `maxSize` at startup, or if the application doesn't log enough new data to trigger an internal size check post-initialization, rotation may not occur as expected.  

> [!WARNING]
> 
> For long-running applications or those with inconsistent logging patterns, this can lead to uncontrolled log file growth.

### Custom Archiving

For more complex archiving logic (like compressing old logs, moving them to a different location, or deleting them based on age), you can use the `log.transports.file.archiveLog` function. This function is called before a log file is overwritten during rotation (if `maxSize` is set and reached) or when `log.rotateLogNow()` is called.

```typescript
log.transports.file.archiveLog = (file) => {
  // Example: Compress the log file
  // const Gzip = zlib.createGzip();
  // const  source = fs.createReadStream(file.path);
  // const  destination = fs.createWriteStream(`${file.path}.gz`);
  // source.pipe(Gzip).pipe(destination);
  // fs.unlinkSync(file.path); // Remove original after archiving
  console.log('Archiving log file:', file.path);
};
```

# Process Specific Logging

> [!NOTE]
> 
> For consistent log organization establish identifiers like `SESSION_ID` or `UTILITY_PROCESS_ID` early in your application's lifecycle. A common approach is to set them as environment variables.

## Logging `MAIN PROCESS` & `RENDER PROCESS(es)`

<details>
<summary><code>MAIN PROCESS</code> Logger</summary>

```javascript
// electron/main/logger/index.ts

import log from 'electron-log/main'
import path from 'path';
import moment from 'moment'

const messageFormat = '{h}:{i}:{s} [{processType}{scope}] [{level}] > {text}';

log.transports.console.format = messageFormat

log.transports.file.format = messageFormat
log.transports.file.resolvePathFn = (variables, message) => {
  /**
   * .../<APP_NAME>/
   *    logs/
   *      YYYY-MM-DD/
   *        <SESSION_ID>
   *          main.log | renderer.log
  */

  // These environment variables should be set when the `MAIN PROCESS` is spawned.
  const SESSION_ID = process.env.SESSION_ID || 'unknown-session';

 const now = moment().utc()
 const year = now.year().toString()
 const month = (now.month() + 1).toString().padStart(2, '0')
 const day = now.date().toString().padStart(2, '0')
  
  const datePath = `${year}-${month}-${day}`;
  const fileName = `${message?.variables?.processType || 'unknown'}.log`;

  return path.join(
    variables.userData,
    'logs',
    datePath,
    SESSION_ID,
    fileName
  );
};
```

</details>
</br>

<details>
<summary><code>RENDER PROCESS(es)</code> Logger</summary>

```ts
// src/lib/logger.ts

import log from 'electron-log/renderer';

// Logs from the electron-log/renderer are automatically sent to the 
// `MAIN PROCESS` logger via IPC communication. This communication is 
// set up when log.initialize() is called in the main process.
// No separate initialization is typically needed here for basic IPC logging.

export default log;
```

</details>
</br>

Key components of the structure:

  - `{APP_NAME}`: The name of the application (`electron_example`).
  - `<YYYY-MM-DD>`: A directory for each day's logs.
  - `<SESSION_ID>`: A directory for each unique session _(each time the application is opened)_.

## Logging `UTILITY PROCESS(es)`

> [!IMPORTANT]
>
> the `UTILITY PROCESS(es)` logging **does not** communicate with the `MAIN PROCESS` logger like the `RENDER PROCESS(es)` loggers do over the `IPC`. The `UTILITY PROCESS(es)` configure their own `electron-log/main` instance.

<details>
<summary><code>UTILITY PROCESS(es)</code>Logger</summary>

```javascript
import log from 'electron-log/main'
import path from 'path';
import moment from 'moment' 

const messageFormat = '{h}:{i}:{s} [{processType}{scope}] [{level}] > {text}';

log.transports.console.format = messageFormat;

log.transports.file.format = messageFormat;
log.transports.file.resolvePathFn = (variables, message) => {
  /**
   * .../<APP_NAME>/
   *  YYYY-MM-DD/
   *    <SESSION_ID>
   *      main.log | renderer.log  (from other processes)
   *      <utility scope>/         (folder for this type of utility)
   *        <UTILITY_PROCESS_ID>/  (folder for this specific utility instance)
   *          utility.log
  */

  // These environment variables should be set when the utility process is spawned.
  const SESSION_ID = process.env.SESSION_ID || 'unknown-session';
  const UTILITY_PROCESS_ID = process.env.UTILITY_PROCESS_ID || 'unknown-utility-instance';
  
  const now = moment().utc()
  const year = now.year().toString()
  const month = (now.month() + 1).toString().padStart(2, '0')
  const day = now.date().toString().padStart(2, '0')
  
  const datePath = `${year}-${month}-${day}`;
  const fileName = `utility.log`;

  return path.join(
    variables.userData,
    'logs',
    datePath,
    SESSION_ID,
    message.scope || 'undefined-utility-scope',
    UTILITY_PROCESS_ID,
    fileName
  );
};
```

</details>

Key components of the structure:

  - `{APP_NAME}`: The name of the application (`electron_example`).
  - `<YYYY-MM-DD>`: A directory for each day's logs.
  - `<SESSION_ID>`: A directory for each unique session _(each time the application is opened)_.
  - `<utility scope>` : Additional log context, can be set via `log.scope('scopeName')` in the utility process.
  - `<UTILITY_PROCESS_ID>`: A directory for each unique instance of a utility process.

# Structured Logging with `electron-log`

While `electron-log`'s default text-based format is convenient for human reading, structured logging (typically in JSON format) is highly beneficial for production environments. JSON logs are easier to parse, search, and analyze by log management systems.

Effective logs include rich contextual information that helps in diagnosing issues (while respecting memory usage & readability / parsing concerns). Consider adding fields like:

  - `appName`: `app.getName()`
  - `appVersion`: `app.getVersion()`
  - `electronVersion`: `process.versions.electron`
  - `chromeVersion`: `process.versions.chrome`
  - `nodeVersion`: `process.versions.node`
  - `osPlatform`: `os.platform()`
  - `osRelease`: `os.release()`
  - `processId`: `process.pid`

Here's an example of incorporating some of this into the custom JSON formatter:

```typescript
log.transports.file.format = (message) => {
  const logData = {
    timestamp: message.date.toISOString(),
    level: message.level,
    processType: message.variables.processType || 'unknown',
    scope: message.scope,
    message: message.data.join(' '),
    context: {
      appName: app.getName(),
      appVersion: app.getVersion(),
      electronVersion: process.versions.electron,
      osPlatform: os.platform(),
      processId: process.pid,
    }
  };
  return JSON.stringify(logData);
};
```

# Development vs. Production Logging

Logging needs often differ between development and production environments.

  - **Development**: You might prefer more verbose logging, extensive console output, and human-readable formats.
  - **Production**: Focus on structured JSON logs written to files, potentially with less verbosity by default, and ensure sensitive data is not logged.

Use Electron's `app.isPackaged` property or environment variables (`NODE_ENV`) to configure logging differently:

```typescript

// available:  error | warn | info | verbose | debug | silly

if (app.isPackaged) {
  // Production logging settings
  log.transports.console.level = 'warn'; // Less console noise
  log.transports.file.level = 'info';
  // log.transports.file.format = ...
  // log.transports.console.format = ...
} else {
  // Development logging settings
  log.transports.console.level = 'debug';
  log.transports.file.level = 'debug';
  // log.transports.file.format = ...
  // log.transports.console.format = ...
}

// Note: electron-log's IPC transport (for main process logs to renderer console)
// is disabled by default in production (when app.isPackaged is true).
```

# Security Considerations

**Never log sensitive information.** This includes passwords, API keys, personally identifiable information (PII), or any other confidential data. Review your log messages and contextual data to ensure they do not inadvertently expose such information.

> [!IMPORTANT]
>
> `electron-log` does not offer built-in features for automatic redaction or masking of sensitive data within log messages. The responsibility for preventing sensitive data leakage into logs falls entirely on the application developer. You must implement custom logic to scrub or mask data *before* it is passed to `electron-log`'s logging methods.

# A Note on Alternative Logging Libraries

While `electron-log` is an excellent choice for most Electron applications due to its simplicity and built-in Electron-specific features like automated IPC for renderer logs, other powerful logging libraries like `Winston` and `Pino` are also available. These libraries offer extensive configurability and performance characteristics that might be beneficial for very complex applications or those with extreme logging volume. However, they typically require more manual setup for Electron's multi-process environment.