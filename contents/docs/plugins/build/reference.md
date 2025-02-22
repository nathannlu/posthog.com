---
title: Plugins Developer Reference
sidebar: Docs
showTitle: true
---

> **Note:** It's worth reading the [Building Plugins Overview](./overview) for a quick introduction to how to build your own plugin.

## plugin.json file

A `plugin.json` file is structured as follows:

```json
{
  "name": "<plugin_display_name>",
  "url": "<repo_url>",
  "description": "<description>",
  "main": "<entry_point>",
  "config": [
    {
      "markdown": "Custom markdown block before the fields,\n[Use links](http://example.com) and other goodies!"
    },
    {
      "key": "param1",
      "name": "<param1_name>",
      "type": "<param1_type>",
      "default": "<param1_default_value>",
      "hint": "<param1_hint_value>",
      "required": true,
      "secret": true
    },
    {
      "name": "<param2_name>",
      "type": "<param2_type>",
      "default": "<param2_default_value>",
      "required": false,
    }
  ]
}
```

Here's an example `plugin.json` file from our ['Hello World Plugin'](https://github.com/PostHog/helloworldplugin):

```json
{
  "name": "Hello World",
  "url": "https://github.com/PostHog/helloworldplugin",
  "description": "Greet the World and Foo a Bar, JS edition!",
  "main": "index.js",
  "config": [
    {
      "markdown": "This is a sample plugin!"
    },
    {
      "key": "bar",
      "name": "What's in the bar?",
      "type": "string",
      "default": "baz",
      "hint": "This will be sent in a **property**",
      "required": false
    }
  ]
}
```

Most options in this file are self-explanatory, but there are a few worth exploring further:

### main

`main` determines the entry point for your plugin, where your `setupPlugin` and `processEvent` functions are. More on these later.

### config

`config` consists of an array of objects that each pertain to a specific configuration field or markdown explanation for your plugin.

Each object in a config can have the following properties:

|   Key    |                    Type                    |                                                                           Description                                                                           |
| :------: | :----------------------------------------: | :-------------------------------------------------------------------------------------------------------------------------------------------------------------: |
|   type   | `"string"` or `"attachment"` or `"choice"` | Determines the type of the field - "attachment" asks the user for an upload, and "choice" requires the config object to have a `choices` array, explained below |
|   key    |                  `string`                  |                                     The key of the plugin config field, used to reference the value from inside the plugin                                      |
|   name   |                  `string`                  |                                          Displayable name of the field - appears on the plugin setup in the PostHog UI                                          |
| default  |                  `string`                  |                                                                   Default value of the field                                                                    |
|   hint   |                  `string`                  |                                             More information about the field, displayed under the in the PostHog UI                                             |
| markdown |                  `string`                  |                                                             Markdown to be displayed with the field                                                             |
|  order   |                  `number`                  |                                                                           Deprecated                                                                            |
| required |                 `boolean`                  |                                               Specifies if the user needs to provide a value for the field or not                                               |
|  secret  |                 `boolean`                  |                     Secret values are write-only and never shown to the user again - useful for plugins that ask for API Keys, for example                      |
| choices  |                  `string[]`                   |                           Only accepted on configs with `type` equal to `"choice"` - an array of choices (of type `string`) to be presented to the user                            |

> **Note:** You can have a config field that only contains `markdown`. This won't be used to configure your plugin but can be placed anywhere in the `config` array and is useful for customizing the content of your plugin's configuration step in the PostHog UI.

## PluginMeta

> Check out [Plugin Types](/docs/plugins/types) for a full spec of types for plugin authors.

**Every plugin server function** is called by the plugin server with an object of type `PluginMeta` that will always contain the object `cache`, and can also include `global`, `attachments`, and `config`, which you can use in your logic. 

Here's what they do:

### config

Gives you access to the plugin config values as described in `plugin.json` and configured via the PostHog interface.

### cache

A way to store values that persist across special function calls. The values are stored in [Redis](https://redis.io/), an in-memory store.

The `cache` type is defined as follows:

```js
interface CacheExtension {
    set: (key: string, value: unknown, ttlSeconds?: number) => Promise<void>
    get: (key: string, defaultValue: unknown) => Promise<unknown>
    incr: (key: string) => Promise<number>
    expire: (key: string, ttlSeconds: number) => Promise<boolean>
}
```

Storing values is done via `cache.set`, which takes a key and a value, as well as an optional value in seconds after which the key will expire.

Retrieving values uses `cache.get`, which takes the key of the value to be retrieved, as well as a default value in case the key does not exist.

You can also use `cache.incr` to increment numerical values by 1, and `cache.expire` to make [keys volatile](https://redis.io/commands/expire), meaning they will expire after the specified number of seconds.


### global

Global is used for sharing functionality between `setupPlugin` and the rest of the special functions, like `processEvent`, `onEvent`, or `runEveryMinute`, since global scope does not work in the context of PostHog plugins. 

### attachments

Attachments gives access to files uploaded by the user for config parameters of type `attachment`. An `attachment` has the following type definition:

```js
interface PluginAttachment {
    content_type: string
    file_name: string
    contents: any
}
```

As such, accessing the contents of an uploaded file can be done with `attachments.attachmentName.contents`.

### jobs

The `jobs` object gives you access to the jobs you specified in your plugin. See [Jobs](#jobs-1) for more information.

## setupPlugin function

`setupPlugin` is a function you can use to dynamically set plugin configuration based on the user's inputs at the configuration step. 

You could, for example, check if an API Key inputted by the user is valid and throw an error if it isn't, prompting PostHog to ask for a new key.

It takes only an object of type `PluginMeta` as a parameter and does not return anything.

Example (from the [PostHog MaxMind Plugin](https://github.com/PostHog/posthog-maxmind-plugin)):

```js
export function setupPlugin({ attachments, global }) {
    if (attachments.maxmindMmdb) {
        global.ipLookup = new Reader(attachments.maxmindMmdb.contents)
    }
}
```

## processEvent function

> If you were using `processEventBatch` before, you should now use `processEvent`. `processEventBatch` has been **deprecated**.

`processEvent` is the juice of your plugin. 

In essence, it takes an event as a parameter and returns an event as a result. In the process, this event can be:

- Modified
- Sent somewhere else
- Not returned (preventing ingestion)

It takes an event and an object of type `PluginMeta` as parameters and returns an event.

Here's an example (from the ['Hello World Plugin'](https://github.com/PostHog/helloworldplugin)):

```js
async function processEvent(event, { config, cache }) {
    const counter = await cache.get('counter', 0)
    cache.set('counter', counter + 1)

    if (event.properties) {
        event.properties['hello'] = 'world'
        event.properties['bar'] = config.bar
        event.properties['$counter'] = counter
    }

    return event
}
```

As you can see, the function receives the event before it is ingested by PostHog, adds properties to it (or modifies them), and returns the enriched event, which will then be ingested by PostHog (after all plugins run).

> Please note that `$snapshot` events (used for session recordings) do not go through `processEvent`. Instead, you can access them via the `onSnapshot` function described below.

## onEvent function

> **Minimum Plugin Server version:** 0.19.0

`onEvent` works similarly to `processEvent`, except any returned value is ignored by the plugin server. In other words, `onEvent` can read an event but not modify it. 

In addition, `onEvent` functions will run after all enabled plugins have run `processEvent`. This ensures you will be receiving an event following all possible modifications to it.

This was originally built for and is particularly useful for export plugins. These plugins need to receive the "final form" of an event and send it out of PostHog, without having to modify it.

Here's a quick example:

```js
async function onEvent(event) {
    // do something to the event
    sendEventToSalesforce(event)

    // no need to return anything
}
```


## onSnapshot function

> **Minimum Plugin Server version:** 0.19.0

`onSnapshot` works exactly like `onEvent`. The only difference between the two is that `onSnapshot` receives session recording events, while `onEvent` receives all other events.

## Scheduled Tasks

Plugins can also run scheduled tasks through the functions:

- `runEveryMinute`
- `runEveryHour`
- `runEveryDay`

These functions only take an object of type `PluginMeta` as a parameter and do not return anything.

Example usage:

```js
async function runEveryMinute({ config }) {
    const url = `https://api.github.com/repos/PostHog/posthog`
    const response = await fetch(url)
    const metrics = await response.json()

  // posthog.capture is also available in plugins by default
    posthog.capture('github metrics', { 
        stars: metrics.stargazers_count,
        open_issues: metrics.open_issues_count,
        forks: metrics.forks_count,
        subscribers: metrics.subscribers_count
    })
}
```

It's worth noting that the plugin server supports debouncing, meaning that the counter for the next task will only start once the previous task finishes. In other words, if a given task that runs "every minute" takes longer than a minute, the next task will only start one minute after the previous task finishes.

## Jobs

> **Minimum Plugin Server version:** 0.18.0

Jobs are a way for plugin developers to schedule and run tasks asynchronously using a powerful scheduling API.

Jobs make possible use cases such as retrying failed requests, a key component of plugins that export data out of PostHog.

### Specifying jobs

To specify jobs, you should export a `jobs` object mapping string keys (job names) to functions (jobs), like so:

```js
export const jobs = {
    retryRequest: (request, meta) => {
        fetch(request.url, request.options)
    }
}
```

Job functions can optionally take a payload as their first argument, which can be of any type. They can also access the `meta` object, which is appended as an argument to all plugin functions, meaning that it will be the second argument in the presence of a payload, and the first (and only) argument in the absence of one.

### Triggering a job

Jobs are accessed as `jobs` via the `meta` object. Triggering a job works as follows:

```js
await jobs.retryRequest(request).runIn(30, 'seconds')
await jobs.retryRequest(request).runNow()
await jobs.retryRequest(request).runAt(new Date())
```

Having gotten a job function via its key from the `jobs` object calling the function with the desired payload will return another object with 3 functions that can be used to schedule your job. They are:

- `runNow`: Runs the job now, but does so asynchronously
- `runAt`: Takes a JavaScript `Date` object that specifies when the job should run
- `runIn`: Takes a duration as a `number` and a unit as a `string` specifying in how many units of time to run this job (e.g. 1 hour)

> Accepted values for the unit argument of `runIn` are: 'milliseconds', 'seconds', 'minutes', 'hours', 'days', 'weeks', 'months', 'quarters', and 'years'. The function will accept these in both singular (e.g. 'second') or plural (e.g. 'seconds') form.

All jobs return a promise that does not resolve to any value. 

### Full example

```js
export const jobs = {
    continueSearchingForTheTeapot: async (request, meta) => {
        await lookForTheTeapot(request)
    }
}

async function lookForTheTeapot (request) {
    const res = await fetch(request.url)
    if (res.status !== 418) {
        await jobs.lookForTheTeapot(request).runIn(30, 'seconds')
        return
    }
    console.log('found the teapot!')
}

export async function processEvent (event, { jobs }) {

    const request = { url: 'https://www.google.com/teapot' }
    await lookForTheTeapot(request)
    
    return event
}
```

## Limitations

PostHog plugins are still in beta, and our scheduled tasks are the newest feature within plugins. As such, they currently have a few limitations:

1. The time intervals (e.g. "every minute" / "every hour") are promises, not guarantees. A worker may be down for 2 seconds because of a restart and miss the task. We're working to add better timing guarantees in the upcoming releases.
