---
title: 基于 Playwright 的课程助手
date: 2026-04-06 12:19:33
updated: 2026-04-06 12:19:33
tags: [Playwright]
---

这篇文章将使用 Node.js 版 [Playwright](https://playwright.dev/) 来完成一个特定网站的在线课程学习任务。

## 课程网站介绍

这个网站是基于 [Vue.js](https://cn.vuejs.org/) 构建的，在网页的源代码中可以发现引入了 *vue.min.js*、*vue-router.min.js* 和 *vuex.min.js* 等文件。这个网站需要使用用户名、密码和手机验证码登录，然后进入课程页面开始学习。课程页面包含课程列表，点击课程进入章节列表页面，点击章节进入视频列表页面，视频是使用 [\<video\>](https://www.w3school.com.cn/tags/tag_video.asp) 标签实现的，一个视频播放完成后会自动播放下一个视频。这些跳转都在一个浏览器页面中完成，因此这个网站是一个单页面应用？

<!-- more -->

## 创建项目和实现脚本

执行 `mkdir course-helper && cd course-helper && npm init` 创建项目并初始化，执行 `npm i -D playwright@1.59.1 @playwright/test@1.59.1` 安装依赖包，修改 *package.json* 文件添加启动脚本 `"start": "node main.js"`。最终的 *package.json* 文件如下：

```json
{
  "name": "course-helper",
  "version": "1.0.0",
  "main": "main.js",
  "scripts": {
    "start": "node main.js"
  },
  "author": "",
  "license": "ISC",
  "description": "",
  "devDependencies": {
    "@playwright/test": "^1.59.1",
    "playwright": "^1.59.1"
  }
}
```

创建 *main.js* 文件并实现脚本：

```javascript
const { chromium } = require("playwright");
const { expect } = require("@playwright/test");

const main = async (cdpAddress, courseUrl, loggedIn, remaining = 10) => {
    let browser, context, page;

    try {
        browser = await chromium.connectOverCDP(cdpAddress);
        context = browser.contexts()[0];
        page = await context.newPage();

        await page.goto(courseUrl);

        await expect(page.getByText(loggedIn)).toBeVisible({ timeout: 10000 });

        const items = await page.locator("ul.by-timeline > li.by-timeline-item").evaluateAll(elements => {
            return elements.map(element => {
                const type = element.querySelector(".type");
                const title = Array.from(element.querySelectorAll(".title-time span")).find(s => !s.className);
                const completeStatus = element.querySelector(".complete-status");
                return {
                    type: (type?.innerText || "").trim(),
                    title: (title?.innerText || "").trim(),
                    completeStatus: (completeStatus?.innerText || "").trim(),
                };
            });
        });

        await page.waitForTimeout(5000);

        for (const item of items) {
            if (item.type === "考试") {
                continue;
            }
            // if (item.completeStatus === "已完成") {
            //     continue;
            // }

            await page.locator("li.by-timeline-item", { hasText: item.title }).first().click();

            // await page.pause();
            await page.waitForTimeout(5000);

            await processCourse(page, remaining);

            await page.goBack({ waitUntil: "networkidle" });
        }
    } catch (error) {
        console.log("发生错误：", error);
    } finally {
        if (page && !page.isClosed()) {
            await page.close();
        }
        if (browser) {
            await browser.close();
        }
    }
};

const processCourse = async (page, remaining = 10) => {
    await expect(page.locator("div.online-detail > div.online-course-item").first()).toBeVisible({ timeout: 10000 });

    const items = await page.locator("div.online-detail > div.online-course-item").evaluateAll(elements => {
        return elements.map(element => {
            const title = element.querySelector(".by-tooltip-directive-reference");
            const completeStatus = element.querySelector(".complete-status");
            return {
                title: (title?.innerText || "").trim(),
                completeStatus: (completeStatus?.innerText || "").trim(),
            };
        });
    });

    for (const item of items) {
        // if (item.completeStatus === "已完成") {
        //     continue;
        // }

        await page.locator("div.online-course-item", { hasText: item.title }).first().click();

        await page.waitForTimeout(5000);

        await playVideos(page, remaining);

        await page.goBack({ waitUntil: "networkidle" });
    }
};

const playVideos = async (page, remaining = 10) => {
    await expect(page.locator("div.course-info-list").locator("ul")).toBeVisible({ timeout: 10000 });

    const items = await page.locator("div.course-info-list").evaluate(element => {
        const spans = element.querySelectorAll("span.course-name");
        return Array.from(spans).map(span => {
            return {
                title: span.innerText.trim(),
            }
        });
    });

    for (const item of items) {
        await page.locator("span.course-name", { hasText: item.title }).first().click();

        await expect(page.locator("video.vjs-tech")).toBeVisible({ timeout: 10000 });

        await page.evaluate(async (remaining) => {
            const video = document.querySelector("video.vjs-tech");

            if (!isFinite(video.duration)) {
                await new Promise(resolve => video.addEventListener("loadedmetadata", () => resolve(), { once: true }));
            }

            video.currentTime = video.duration - remaining;

            await new Promise(resolve => video.addEventListener("canplay", () => resolve(), { once: true }));

            video.play();

            await new Promise(resolve => video.addEventListener("ended", () => resolve(), { once: true }));
        }, remaining);
    }
}

const isValidUrl = (string) => {
    try {
        const url = new URL(string);
        return url.protocol === "http:" || url.protocol === "https:";
    } catch {
        return false;
    }
};

const cdpAddress = process.argv[2];
const courseUrl = process.argv[3];
const loggedIn = process.argv[4];
const remaining = process.argv[5] ? parseInt(process.argv[5]) : 10;

const errors = [];
if (!cdpAddress || !isValidUrl(cdpAddress)) {
    errors.push(!cdpAddress ? "缺少 CDP 地址" : "CDP 地址格式错误");
}
if (!courseUrl || !isValidUrl(courseUrl)) {
    errors.push(!courseUrl ? "缺少课程地址" : "课程地址格式错误");
}
if (!loggedIn) {
    errors.push("缺少登录状态确认值");
}
if (errors.length > 0) {
    console.log("用法一: node main.js <CDP地址> <课程地址> <登录状态确认值> [剩余秒数]");
    console.log("用法二: npm run start -- <CDP地址> <课程地址> <登录状态确认值> [剩余秒数]");
    console.log("错误:");
    errors.forEach(error => console.log(`  - ${error}`));
    process.exit(1);
}

main(cdpAddress, courseUrl, loggedIn, remaining);
```

## 脚本解析

### 连接本地浏览器

一开始的想法是使用 [readline](https://nodejs.org/api/readline.html) 从控制台读取用户名、密码，然后获取手机验证码并在控制台输入，即从最初的登录开始实现脚本。这种方式在调试阶段很不方便，而 [connectOverCDP](https://playwright.dev/docs/api/class-browsertype#browser-type-connect-over-cdp) 可以[连接本地浏览器复用登录状态](https://mp.weixin.qq.com/s/FabEYrl6T0wczClE1EbljA)，只需要在本地浏览器登录一次就好了，其中 CDP 是 [Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/) 的简写。要使用 CDP 需要[为浏览器配置启动参数](https://juejin.cn/post/7373505141489729547) `--remote-debugging-port=9222`，端口可以任意指定。下面以 Edge 浏览器为例

{% asset_img edge-remote-debug-port.png %}

配置完成后重启浏览器，在浏览器中输入 [http://localhost:9222/json/version](http://localhost:9222/json/version) 检查是否配置成功，成功时会看到类似下面的结果

```json
{
   "Browser": "Edg/146.0.3856.97",
   "Protocol-Version": "1.3",
   "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36 Edg/146.0.0.0",
   "V8-Version": "14.6.22.11",
   "WebKit-Version": "537.36 (@79f6730cd02cfe9f19d33d1db200bfcdea57cf21)",
   "webSocketDebuggerUrl": "ws://localhost:9222/devtools/browser/c76e743f-9595-42c2-aa56-21a1bec5269d"
}
```

在示例配置下 CDP 地址为 [http://127.0.0.1:9222](http://127.0.0.1:9222)。

**重要提示**：根据[更改了远程调试开关以提高安全性](https://developer.chrome.com/blog/remote-debugging-port?hl=zh-cn)的描述，高版本的 Chromium 浏览器 `--remote-debugging-port` 需要和 `--user-data-dir` 搭配使用。

注意脚本的第 9 行代码，新打开的浏览器 [contexts](https://playwright.dev/docs/api/class-browser#browser-contexts) 的长度为 0，但是我们肯定会登录一次，因此这里获取的是默认的 context。

### 判断登录状态

未登录的网站和已登录的网站显示的内容是不同的，因此脚本的第 14 行代码根据网页上的内容判断登录状态，这个内容应该是唯一的，不然脚本会报错。

### 获取课程列表

脚本第 16\~27 行代码获取课程列表信息，传给 [Locator#evaluateAll](https://playwright.dev/docs/api/class-locator#locator-evaluate-all) 的函数虽然定义在 Node.js 脚本中，但是它**运行在浏览器**中，因此不能使用 Node.js 的模块，只能使用浏览器的 API。第 31~47 行代码依次处理每一个课程。后面的章节和视频都采用这种先获取列表再依次处理每一项的思路。

### 播放视频

传递给 [Page#evaluate](https://playwright.dev/docs/api/class-page#page-evaluate) 的函数也是在浏览器中执行，视频的剩余时间要通过 `evaluate` 的第 2 个参数来传递，参考第 120 行代码。

视频是异步加载的，因此要监听 `loadedmetadata` 事件来等待 `duration` 属性被设置，参考第 110 行代码。在第 113 行代码设置播放时间时，要等待视频加载完成，此时视频可能还不能播放，因此要监听 `canplay` 事件，参考第 115 行代码。

因为会自动播放下一个视频，因此要监听 `ended` 事件，参考第 119 行代码，然后将控制权交回 Playwright 进行下一个视频的播放。在这里走了一些弯路，一开始的思路是判断 `video.ended` 属性，但是在网站自动播放配置下可能错过值为 `true` 的时刻，导致 `evaluate` 超时，因此只能监听 `ended` 事件。

## 遇到的问题

### 视频播放结束后没有返回上一个页面

在其他操作系统和浏览器环境中遇到了最后一个视频播放结束后第 106 行的 `page.evaluate` 没有返回，并且程序没有报错也没有退出，这可能与[事件循环（Event Loop）](https://mdn.org.cn/en-US/docs/Web/JavaScript/Event_loop)和 `evaluate` 的退出机制有关，没有找到具体原因，但尝试了两种可能的解决方法，能一定程度解决问题。

方法一：在第 120 行代码后面添加 `await page.waitForTimeout(1000);`，等待 1 秒后再处理下一个视频。

方法二：修改第 119 行代码，增加超时限制

```javascript
await Promise.race([
    new Promise(resolve => video.addEventListener("ended", () => resolve(), { once: true })),
    new Promise(resolve => setTimeout(() => resolve(), remaining + 5000)),
]);
```
