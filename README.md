# headless-chrome-crawler [![npm](https://badge.fury.io/js/headless-chrome-crawler.svg)](https://www.npmjs.com/package/headless-chrome-crawler) [![build](https://circleci.com/gh/yujiosaka/headless-chrome-crawler/tree/master.svg?style=shield&circle-token=ba45f930aed7057b79f2ac09df6be3e1b8ee954b)](https://circleci.com/gh/yujiosaka/headless-chrome-crawler/tree/master) [![Greenkeeper badge](https://badges.greenkeeper.io/yujiosaka/headless-chrome-crawler.svg)](https://greenkeeper.io/)
Headless Chrome crawls with [jQuery](https://jquery.com) support, powered by [Puppeteer](https://github.com/GoogleChrome/puppeteer)

## Features

Crawlers based on simple requests to HTML files are generally fast. However, it sometimes ends up capturing empty bodies, especially when the websites are built on such modern frontend frameworks as [AngularJS](https://angularjs.org), [React](https://reactjs.org) and [Vue.js](https://jp.vuejs.org/index.html).

Powered by [Puppeteer](https://github.com/GoogleChrome/puppeteer), headless-chrome-crawler provides [simple APIs](#api-reference) to manupluate Headless Chrome and allows you to crawl these dynamic websites with the following features:

* Configure concurrency, delay and retry
* Breadth-first search (BFS) to automatically follow links
* Pluggable cache such as [Redis](https://redis.io) to skip duplicate requests
* Pause at the max request and resume at any time
* Insert [jQuery](https://jquery.com) automatically for scraping
* Save screenshot for the crawling evidence
* Emulate device and user agent
* Priority queue for crawling efficiency
* Promise support

## Getting Started

### Installation

```
yarn add headless-chrome-crawler
# or "npm i headless-chrome-crawler"
```

> **Note**: headless-chrome-crawler is powered by [Puppeteer](https://github.com/GoogleChrome/puppeteer). While installation, it automatically downloads a recent version of Chromium. To skip the download, see [Environment variables](https://github.com/GoogleChrome/puppeteer/blob/master/docs/api.md#environment-variables).

### Usage

The basic API of headless-chrome-crawler is inspired by that of [node-crawler](https://github.com/bda-research/node-crawler), so the API design is somewhat similar but not exactly compatible.

```js
const HCCrawler = require('headless-chrome-crawler');

HCCrawler.launch({
  // Function to be evaluated in browsers
  evaluatePage: (() => ({
    title: $('title').text(),
  })),
  // Function to be called with evaluated results from browsers
  onSuccess: (result => {
    console.log(result);
  }),
})
  .then(crawler => {
    // Queue a request
    crawler.queue('https://example.com/');
    // Queue multiple requests
    crawler.queue(['https://example.net/', 'https://example.org/']);
    // Queue a request with custom options
    crawler.queue({
      url: 'https://www.example.com/',
      // Disable jQuery only for this request
      jQuery: false,
      // Override an already defined evaluatePage option
      evaluatePage: (() => ({
        title: document.title,
      })),
      // Emulate a tablet device
      device: 'Nexus 7',
      // Enable screenshot by passing options
      screenshot: {
        path: './tmp/www-example-com.png'
      },
    });
    crawler.onIdle() // Resolved when no queue is left
      .then(() => crawler.close()); // Close the crawler
  });
```

## Examples

* [Priority queue for crawling efficiency](https://github.com/yujiosaka/headless-chrome-crawler/blob/master/examples/priority-queue.js)
* [Pause at the max request and resume at any time](https://github.com/yujiosaka/headless-chrome-crawler/blob/master/examples/pause-resume.js)
* [Emulate device and user agent](https://github.com/yujiosaka/headless-chrome-crawler/blob/master/examples/emulate-device.js)
* [Redis cache to skip duplicate requests](https://github.com/yujiosaka/headless-chrome-crawler/blob/master/examples/redis-cache.js)
* [Conditionally saving screenshots](https://github.com/yujiosaka/headless-chrome-crawler/blob/master/examples/conditional-screenshot.js)

See [here](https://github.com/yujiosaka/headless-chrome-crawler/tree/master/examples) for the full examples list. The examples can be run from the root folder as follows:

```sh
NODE_PATH=../ node examples/priority-queue.js
```

## API reference

### Table of Contents

* [class: HCCrawler](#class-hccrawler)
  * [HCCrawler.connect([options])](#hccrawlerconnectoptions)
  * [HCCrawler.launch([options])](#hccrawlerlaunchoptions)
  * [HCCrawler.executablePath()](#hccrawlerexecutablepath)
  * [crawler.queue([options])](#crawlerqueueoptions)
  * [crawler.setMaxRequest(maxRequest)](#crawlersetmaxrequestmaxrequest)
  * [crawler.pause()](#crawlerpause)
  * [crawler.resume()](#crawlerresume)
  * [crawler.clearCache()](#crawlerclearcache)
  * [crawler.close()](#crawlerclose)
  * [crawler.disconnect()](#crawlerdisconnect)
  * [crawler.version()](#crawlerversion)
  * [crawler.wsEndpoint()](#crawlerwsendpoint)
  * [crawler.onIdle()](#crawleronidle)
  * [crawler.isPaused](#crawlerispaused)
  * [crawler.queueSize](#crawlerqueuesize)
  * [crawler.pendingQueueSize](#crawlerpendingqueuesize)
  * [crawler.requestedCount](#crawlerrequestedcount)
* [class: SessionCache](#class-sessioncache)
* [class: RedisCache](#class-rediscache)
* [class: BaseCache](#class-basecache)

### class: HCCrawler

HCCrawler provides methods to launch or connect to a HeadlessChrome/Chromium.

```js
const HCCrawler = require('headless-chrome-crawler');

HCCrawler.launch({
  evaluatePage: (() => ({
    title: $('title').text(),
  })),
  onSuccess: (result => {
    console.log(result);
  }),
})
  .then(crawler => {
    crawler.queue('https://example.com/');
    crawler.onIdle()
      .then(() => crawler.close());
  });
```

#### HCCrawler.connect([options])

* `options` <[Object]>
  * `maxConcurrency` <[number]> Maximum number of pages to open concurrently, defaults to `10`.
  * `maxRequest` <[number]> Maximum number of requests, defaults to `0`. Pass `0` to disable the limit.
  * `cache` <[Cache]> A cache object which extends [BaseCache](#class-basecache)'s interfaces to remember and skip duplicate requests, defaults to a [SessionCache](#class-sessioncache) object. Pass `null` if you don't want to skip duplicate requests.
  * `persistCache` <[boolean]> Whether to persist cache on closing or disconnecting from the browser, defaults to `false`.
* returns: <Promise<HCCrawler>> Promise which resolves to HCCrawler instance.

This method connects to an existing Chromium instance. The following options are passed to [puppeteer.connect()](https://github.com/GoogleChrome/puppeteer/blob/master/docs/api.md#puppeteerconnectoptions).

```
browserWSEndpoint, ignoreHTTPSErrors
```

Also, the following options can be set as default values when [crawler.queue()](#crawlerqueueoptions) are executed.

```
url, allowedDomains, timeout, priority, delay, retryCount, retryDelay, jQuery, device, username, password, shouldRequest, evaluatePage, onSuccess, onError
```

> **Note**: In practice, setting the options every time you queue equests is not only redundant but also slow. Therefore, it's recommended to set the default values and override them depending on the necessity.

#### HCCrawler.launch([options])

* `options` <[Object]>
  * `maxConcurrency` <[number]> Maximum number of pages to open concurrently, defaults to `10`.
  * `maxRequest` <[number]> Maximum number of requests, defaults to `0`. Pass `0` to disable the limit.
  * `cache` <[Cache]> A cache object which extends [BaseCache](#class-basecache)'s interfaces to remember and skip duplicate requests, defaults to a [SessionCache](#class-sessioncache) object. Pass `null` if you don't want to skip duplicate requests.
  * `persistCache` <[boolean]> Whether to clear cache on closing or disconnecting from the browser, defaults to `false`.
* returns: <Promise<HCCrawler>> Promise which resolves to HCCrawler instance.

The method launches a HeadlessChrome/Chromium instance. The following options are passed to [puppeteer.launch()](https://github.com/GoogleChrome/puppeteer/blob/master/docs/api.md#puppeteerlaunchoptions).

```
ignoreHTTPSErrors, headless, executablePath, slowMo, args, handleSIGINT, handleSIGTERM, handleSIGHUP, timeout, dumpio, userDataDir, env, devtools
```

Also, the following options can be set as default values when [crawler.queue()](#crawlerqueueoptions) are executed.

```
url, allowedDomains, timeout, priority, delay, retryCount, retryDelay, jQuery, device, username, password, shouldRequest, evaluatePage, onSuccess, onError
```

> **Note**: In practice, setting the options every time you queue the requests is not only redundant but also slow. Therefore, it's recommended to set the default values and override them depending on the necessity.

#### HCCrawler.executablePath()

* returns: <string> An expected path to find bundled Chromium.

See [puppeteer.executablePath()](https://github.com/GoogleChrome/puppeteer/blob/master/docs/api.md#puppeteerexecutablepath) for more details.

#### crawler.queue([options])

* `options` <[Object]>
  * `url` <[String]> Url to navigate to. The url should include scheme, e.g. `https://`.
  * `maxDepth` <[number]> Maximum depth for the crawler to follow links automatically, default to 1. Leave default to disable following links.
  * `priority` <[number]> Basic priority of queues, defaults to `1`. Priority with larger number is preferred.
  * `allowedDomains` <[Array]> List of domains allowed to request. `www.example.com` will be allowed if `example.com` is listed.
  * `delay` <[number]> Number of milliseconds after each request, defaults to `0`. When delay is set, `maxConcurrency` option must be `1`.
  * `retryCount` <[number]> Number of limit when retry fails, defaults to `3`.
  * `retryDelay` <[number]> Number of milliseconds after each retry fails, defaults to `10000`.
  * `jQuery` <[boolean]> Whether to automatically add [jQuery](https://jquery.com) tag to page, defaults to `true`.
  * `device` <[String]> Device to emulate. Available devices are listed [here](https://github.com/GoogleChrome/puppeteer/blob/master/DeviceDescriptors.js).
  * `username` <[String]> Username for basic authentication. pass `null` if it's not necessary.
  * `screenshot` <[Object]> Screenshot option, defaults to `null`. This option is passed to [Puppeteer's page.screenshot()](https://github.com/GoogleChrome/puppeteer/blob/master/docs/api.md#pagescreenshotoptions). Pass `null` or leave default to disable screenshot.
  * `password` <[String]> Password for basic authentication. pass `null` if it's not necessary.
  * `userAgent` <[String]> User agent string to override in this page.
  * `extraHeaders` <[Object]> An object containing additional headers to be sent with every request. All header values must be strings.
  * `preRequest(options)` <[Function]> Function to do anything like modifying `options` before each request. You can also return `false` if you want to skip the request.
    * `options` <[Object]> [crawler.queue()](#crawlerqueueoptions)'s options with default values.
  * `evaluatePage()` <[Function]> Function to be evaluated in browsers. Return serializable object. If it's not serializable, the result will be `undefined`.
  * `onSuccess(response)` <[Function]> Function to be called when `evaluatePage()` successes.
    * `response` <[Object]>
      * `response` <[Object]>
        * `ok` <[boolean]> whether the status code in the range 200-299 or not.
        * `status` <[String]> status code of the request.
        * `url` <[String]> Last requested url.
        * `headers` <[Object]> Response headers.
      * `options` <[Object]> [crawler.queue()](#crawlerqueueoptions)'s options with default values.
      * `result` <[Serializable]> The result resolved from `evaluatePage()` option.
      * `screenshot` <[Buffer]> Buffer with the screenshot image, which is `null` when `screenshot` option not passed.
      * `links` <[Array]> List of links found in the requested page.
  * `onError(error)` <[Function]> Function to be called when request fails.
    * `error` <[Error]> Error object.

> **Note**: `response.url` may be different from `options.url` especially when the requested url is redirected.

The following options are passed to [Puppeteer's page.goto()](https://github.com/GoogleChrome/puppeteer/blob/master/docs/api.md#pagegotourl-options)'s options'.

```
timeout, waitUntil
```

The options can be either an object, an array, or a string. When it's an array, each item in the array will be executed. When it's a string, the options are transformed to an object with only url defined.

#### crawler.setMaxRequest(maxRequest)

This method allows you to modify `maxRequest` option you passed to [HCCrawler.connect()](#hccrawlerconnectoptions) or [HCCrawler.launch()](#hccrawlerlaunchoptions).

#### crawler.pause()

This method pauses processing queues. You can resume the queue by calling [crawler.resume()](#crawlerresume).

#### crawler.resume()

This method resumes processing queues. This method may be used after the crawler is intentionally closed by calling [crawler.pause()](#crawlerpause) or request count reached `maxRequest` option.

#### crawler.clearCache()

returns: <[Promise]> Promise resolved when the cache is cleared.

This method clears the cache when it's used.

#### crawler.close()

returns: <[Promise]> Promise resolved when ther browser is closed.

See [Puppeteer's browser.close()](https://github.com/GoogleChrome/puppeteer/blob/master/docs/api.md#browserclose) for more details.

#### crawler.disconnect()

returns: <[Promise]> Promise resolved when ther browser is disconnected.

See [Puppeteer's browser.disconnect()](https://github.com/GoogleChrome/puppeteer/blob/master/docs/api.md#browserdisconnect) for more details.

#### crawler.version()

returns: <[Promise]> Promise resolved with HeadlessChrome/Chromium version.

See [Puppeteer's browser.version()](https://github.com/GoogleChrome/puppeteer/blob/master/docs/api.md#browserversion) for more details.

#### crawler.wsEndpoint()

returns: <[Promise]> Promise resolved with websocket url.

See [Puppeteer's browser.wsEndpoint()](https://github.com/GoogleChrome/puppeteer/blob/master/docs/api.md#browserwsendpoint) for more details.

#### crawler.onIdle()

- returns: <[Promise]> Promise resolved when queues become empty or paused.

#### crawler.isPaused

* returns: <[boolean]> Whether the queue is paused. This property is read only.

#### crawler.queueSize

* returns: <[number]> The size of queues. This property is read only.

#### crawler.pendingQueueSize

* returns: <[number]> The size of pending queues. This property is read only.

#### crawler.requestedCount

* returns: <[number]> The count of total requests. This property is read only.

### class: SessionCache

`SessionCache` is the [HCCrawler.connect()](#hccrawlerconnectoptions)'s default `cache` option. By default, the crawler remembers already requested urls on its memory. Pass `null` to the option in order to disable it.

```js
const HCCrawler = require('headless-chrome-crawler');

// Pass null to the cache option to disable it.
HCCrawler.launch({ cache: null });
// ...
```

### class: RedisCache

Passing a `RedisCache` object to the [HCCrawler.connect()](#hccrawlerconnectoptions)'s `cache` options allows you to persist requested urls in Redis and prevents from requesting same urls in a distributed servers' environment. It also works well with its `persistCache` option to be true.

Its constructing options are passed to [NodeRedis's redis.createClient()](https://github.com/NodeRedis/node_redis#rediscreateclient)'s options.

```js
const HCCrawler = require('headless-chrome-crawler');
const RedisCache = require('headless-chrome-crawler/cache/redis');

const cache = new RedisCache({ host: '127.0.0.1', port: 6379 });

HCCrawler.launch({
  persistCache: true, // Set true so that cache won't be cleared when closing the crawler
  cache,
});
// ...
```

### class: BaseCache

You can create your own cache by extending the [BaseCache's interfaces](https://github.com/yujiosaka/headless-chrome-crawler/blob/master/cache/base.js) and pass its object to the [HCCrawler.connect()](#hccrawlerconnectoptions)'s `cache` options.

Here is an example of creating a file based cache.

```js
const fs = require('fs');
const HCCrawler = require('headless-chrome-crawler');
const BaseCache = require('headless-chrome-crawler/cache/base');

const FILE = './tmp/fs-cache.json';

// Create a new cache by extending BaseCache interface
class FsCache extends BaseCache {
  init() {
    fs.writeFileSync(this._settings.file, '{}');
    return Promise.resolve();
  }
  clear() {
    fs.unlinkSync(this._settings.file);
    return Promise.resolve();
  }
  close() {
    return Promise.resolve();
  }
  exists(key) {
    const obj = JSON.parse(fs.readFileSync(this._settings.file));
    return Promise.resolve(obj[key] || false);
  }
  set(key) {
    const obj = JSON.parse(fs.readFileSync(this._settings.file));
    obj[key] = true;
    fs.writeFileSync(this._settings.file, JSON.stringify(obj));
    return Promise.resolve();
  }
  remove(key) {
    const obj = JSON.parse(fs.readFileSync(this._settings.file));
    delete obj[key];
    fs.writeFileSync(FILE, JSON.stringify(obj));
    return Promise.resolve();
  }
}

HCCrawler.launch({ cache: new FsCache({ file: FILE }) });
// ...
```

## Tips

### Launch options

[HCCrawler.launch()](#hccrawlerlaunchoptions)'s options are passed to [puppeteer.launch()](https://github.com/GoogleChrome/puppeteer/blob/master/docs/api.md#puppeteerlaunchoptions). It may be useful to set the `headless` and `slowMo` options so that you can see what is going on.

```js
HCCrawler.launch({ headless: false, slowMo: 10 });
```

Also, the `args` option is passed to the browser instance. List of Chromium flags can be found [here](http://peter.sh/experiments/chromium-command-line-switches/). Passing `--disable-web-security` flag is useful for crawling. If the flag is set, links within iframes are collected as those of parent frames. If it's not, the source attributes of the iframes are collected as links.

```js
HCCrawler.launch({ args: ['--disable-web-security'] });
```

### Enable debug logging

All requests and browser's logs are logged via the [debug](https://github.com/visionmedia/debug) module under the `hccrawler` namespace.

```
env DEBUG="hccrawler:*" node script.js
env DEBUG="hccrawler:request" node script.js
env DEBUG="hccrawler:browser" node script.js
```

## FAQ

### How is this different from other crawlers?

There are roughly two types of crawlers. One is static and the other is dynamic.

The static crawlers are based on simple requests to HTML files. They are generally fast, but fail scraping the contents when the HTML dynamically changes on browsers.

Dynamic crawlers based on [PhantomJS](http://phantomjs.org) and [Selenium](http://www.seleniumhq.org) work magically on such dynamic applications. However, [PhantomJS's maintainer has stepped down and recommended to switch to Headless Chrome](https://groups.google.com/forum/#!topic/phantomjs/9aI5d-LDuNE), which is fast and stable. [Selenium](http://www.seleniumhq.org) is still a well-maintained cross browser platform which runs on Chrome, Safari, IE and so on. However, crawlers do not need such cross browsers support.

 This crawler is dynamic and based on Headless Chrome.
