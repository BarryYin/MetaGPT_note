# 第四章 OSS订阅智能体实现

本节课主要介绍如何利用MetaGPT实现一个OSS订阅智能体实现，分别有以下几个模块：

1、爬取数据
2、清洗数据
3、总结结论
4、定时器
5、推送端口

整体而言，算是一个LLM + Agent + RPA的工程实例。从效果上实现信息的自动抓取，并进行推送。但这里用到的RAC的部分并不多，主要是利用了MetaGPT的SOP的能力，可以实现一个简单的爬虫。

在案例代码部分的坑太多了，可能是迭代升级了基础架构，但没有升级案例，总之跑了2小时，没通。希望能给一个跑通无误的案例或者说明吧。

目前卡点问题是：

    raise client_error(req.connection_key, exc) from exc
aiohttp.client_exceptions.ClientProxyConnectionError: Cannot connect to host 127.0.0.1:8118 ssl:default [Connect call failed ('127.0.0.1', 8118)]



----------

目前代码如下：

import asyncio
import os
import time
from typing import Any, AsyncGenerator, Awaitable, Callable, Optional

import aiohttp
import discord
from aiocron import crontab
from bs4 import BeautifulSoup
from pydantic import BaseModel, Field
from pytz import BaseTzInfo

from metagpt.actions.action import Action
from metagpt.config import CONFIG
from metagpt.logs import logger
from metagpt.roles import Role
from metagpt.schema import Message
from metagpt.subscription import SubscriptionRunner

# Actions 的实现
TRENDING_ANALYSIS_PROMPT = """# Requirements
You are a GitHub Trending Analyst, aiming to provide users with insightful and personalized recommendations based on the latest
GitHub Trends. Based on the context, fill in the following missing information, generate engaging and informative titles, 
ensuring users discover repositories aligned with their interests.

# The title about Today's GitHub Trending
## Today's Trends: Uncover the Hottest GitHub Projects Today! Explore the trending programming languages and discover key domains capturing developers' attention. From ** to **, witness the top projects like never before.
## The Trends Categories: Dive into Today's GitHub Trending Domains! Explore featured projects in domains such as ** and **. Get a quick overview of each project, including programming languages, stars, and more.
## Highlights of the List: Spotlight noteworthy projects on GitHub Trending, including new tools, innovative projects, and rapidly gaining popularity, focusing on delivering distinctive and attention-grabbing content for users.
---
# Format Example

```
# [Title]

## Today's Trends
Today, ** and ** continue to dominate as the most popular programming languages. Key areas of interest include **, ** and **.
The top popular projects are Project1 and Project2.

## The Trends Categories
1. Generative AI
    - [Project1](https://github/xx/project1): [detail of the project, such as star total and today, language, ...]
    - [Project2](https://github/xx/project2): ...
...

## Highlights of the List
1. [Project1](https://github/xx/project1): [provide specific reasons why this project is recommended].
...
```

---
# Github Trending
{trending}
"""


class CrawlOSSTrending(Action):
    async def run(self, url: str = "https://github.com/trending"):
        async with aiohttp.ClientSession() as client:
            async with client.get(url, proxy=CONFIG.global_proxy) as response:
                response.raise_for_status()
                html = await response.text()

        soup = BeautifulSoup(html, "html.parser")

        repositories = []

        for article in soup.select("article.Box-row"):
            repo_info = {}

            repo_info["name"] = (
                article.select_one("h2 a")
                .text.strip()
                .replace("\n", "")
                .replace(" ", "")
            )
            repo_info["url"] = (
                "https://github.com" + article.select_one("h2 a")["href"].strip()
            )

            # Description
            description_element = article.select_one("p")
            repo_info["description"] = (
                description_element.text.strip() if description_element else None
            )

            # Language
            language_element = article.select_one(
                'span[itemprop="programmingLanguage"]'
            )
            repo_info["language"] = (
                language_element.text.strip() if language_element else None
            )

            # Stars and Forks
            stars_element = article.select("a.Link--muted")[0]
            forks_element = article.select("a.Link--muted")[1]
            repo_info["stars"] = stars_element.text.strip()
            repo_info["forks"] = forks_element.text.strip()

            # Today's Stars
            today_stars_element = article.select_one(
                "span.d-inline-block.float-sm-right"
            )
            repo_info["today_stars"] = (
                today_stars_element.text.strip() if today_stars_element else None
            )

            repositories.append(repo_info)

        return repositories


class AnalysisOSSTrending(Action):
    async def run(self, trending: Any):
        return await self._aask(TRENDING_ANALYSIS_PROMPT.format(trending=trending))


# Role实现
class OssWatcher(Role):
    def __init__(
        self,
        name="Codey",
        profile="OssWatcher",
        goal="Generate an insightful GitHub Trending analysis report.",
        constraints="Only analyze based on the provided GitHub Trending data.",
    ):
        super().__init__()
        self._init_actions([CrawlOSSTrending, AnalysisOSSTrending])
        self._set_react_mode(react_mode="by_order")

    async def _act(self) -> Message:
        logger.info(f"{self._setting}: ready to {self.rc.todo}")
        # By choosing the Action by order under the hood
        # todo will be first SimpleWriteCode() then SimpleRunCode()
        todo = self.rc.todo

        msg = self.get_memories(k=1)[0]  # find the most k recent messages
        result = await todo.run(msg.content)

        msg = Message(content=str(result), role=self.profile, cause_by=type(todo))
        self.rc.memory.add(msg)
        return msg


# Trigger
class OssInfo(BaseModel):
    url: str
    timestamp: float = Field(default_factory=time.time)

    def __str__(self):
        return f"{self.url}/timestamp={self.timestamp}"


class GithubTrendingCronTrigger:
    def __init__(
        self,
        spec: str,
        tz: Optional[BaseTzInfo] = None,
        url: str = "https://github.com/trending",
    ) -> None:
        self.crontab = crontab(spec, tz=tz)
        self.url = url

    def __aiter__(self):
        return self

    async def __anext__(self):
        await self.crontab.next()
        #return Message(OssInfo(url=self.url))
        print(OssInfo(url=str(self.url)))
        print(str(OssInfo(url=str(self.url))))
        #print(Message(OssInfo(url=self.url)))
        return Message(str(OssInfo(url=str(self.url))))


# callback
async def discord_callback(msg: Message):
    intents = discord.Intents.default()
    intents.message_content = True
    client = discord.Client(intents=intents, proxy=CONFIG.global_proxy)
    token = os.environ["DISCORD_TOKEN"]
    channel_id = int(os.environ["DISCORD_CHANNEL_ID"])
    async with client:
        await client.login(token)
        channel = await client.fetch_channel(channel_id)
        lines = []
        for i in msg.content.splitlines():
            if i.startswith(("# ", "## ", "### ")):
                if lines:
                    await channel.send("\n".join(lines))
                    lines = []
            lines.append(i)

        if lines:
            await channel.send("\n".join(lines))


class WxPusherClient:
    def __init__(
        self,
        token: Optional[str] = None,
        base_url: str = "http://wxpusher.zjiecode.com",
    ):
        self.base_url = base_url
        self.token = token or os.environ["WXPUSHER_TOKEN"]

    async def send_message(
        self,
        content,
        summary: Optional[str] = None,
        content_type: int = 1,
        topic_ids: Optional[list[int]] = None,
        uids: Optional[list[int]] = None,
        verify: bool = False,
        url: Optional[str] = None,
    ):
        payload = {
            "appToken": self.token,
            "content": content,
            "summary": summary,
            "contentType": content_type,
            "topicIds": topic_ids or [],
            "uids": uids or os.environ["WXPUSHER_UIDS"].split(","),
            "verifyPay": verify,
            "url": url,
        }
        url = f"{self.base_url}/api/send/message"
        return await self._request("POST", url, json=payload)

    async def _request(self, method, url, **kwargs):
        async with aiohttp.ClientSession() as session:
            async with session.request(method, url, **kwargs) as response:
                response.raise_for_status()
                return await response.json()


async def wxpusher_callback(msg: Message):
    client = WxPusherClient()
    await client.send_message(msg.content, content_type=3)


# 运行入口，
async def main(spec: str = "04 11 * * *", discord: bool = True, wxpusher: bool = False):
    callbacks = []
    if discord:
        callbacks.append(discord_callback)

    if wxpusher:
        callbacks.append(wxpusher_callback)

    if not callbacks:

        async def _print(msg: Message):
            print(msg.content)

        callbacks.append(_print)

    async def callback(msg):
        await asyncio.gather(*(call(msg) for call in callbacks))

    runner = SubscriptionRunner()
    await runner.subscribe(OssWatcher(), GithubTrendingCronTrigger(spec), callback)
    await runner.run()


if __name__ == "__main__":
    import fire

    fire.Fire(main)



关于作业部分，目前没有时间做，先放在这，跑通了案例，再说。

- 根据前面你所学习的爬虫基本知识（如果你对写爬虫代码感到不熟练，使用GPT帮助你），为你的Agent自定义两个获取资讯的Action类
  - Action 1：根据第四章 3.2.1和3.2.2的指引，独立实现对Github Trending(https://github.com/trending)页面的爬取，并获取每一个项目的 名称、URL链接、描述
  - Action 2：独立完成对Huggingface Papers（https://huggingface.co/papers）页面的爬取，先获取到每一篇Paper的链接（提示：标题元素中的href标签），并通过链接访问标题的描述页面（例如：https://huggingface.co/papers/2312.03818），在页面中获取一篇Paper的 标题、摘要
- 参考第三章 1.4 的内容，重写有关方法，使你的Agent能自动生成总结内容的目录，然后根据二级标题进行分块，每块内容做出对应的总结，形成一篇资讯文档；
- 自定义Agent的SubscriptionRunner类，独立实现Trigger、Callback的功能，让你的Agent定时为通知渠道发送以上总结的资讯文档（尝试实现邮箱发送的功能，这是加分项）

