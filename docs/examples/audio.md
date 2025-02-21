# 示例：下载指定歌单所有歌曲

```python
from bilibili_api import audio, sync, get_session
import os
from curl_cffi import requests

AUDIO_LIST_ID = 10624


async def main():
    # 1. 获取歌单歌曲列表

    al = audio.AudioList(AUDIO_LIST_ID)  # 初始化歌单对象
    audios = []  # 音频对象列表
    p = 1  # 记录目前读取到的页码

    while True:
        l = await al.get_song_list(p)
        audios.extend(l["data"])  # 将当前页的音频全部加入音频对象列表
        if l["pageCount"] >= p:  # 当页码大于总页数 pageCount 停止
            break

        p += 1  # 抓取下一页

    # 2. 下载音频
    sess: requests.AsyncSession = (
        get_session()
    )  # bilibili-api 默认使用 curl_cffi 作为请求客户端

    # 创建歌单文件夹
    if not os.path.exists(str(AUDIO_LIST_ID)):
        os.mkdir(str(AUDIO_LIST_ID))

    for au in audios:
        # 获取下载链接
        a = audio.Audio(au["id"])
        url = await a.get_download_url()
        # url -> {..., 'cdns': ['https://upos-sz-mirrorhw.bilivideo.com/...'], ...}
        url = url["cdns"][0]
        # 下载歌曲
        file = f"{AUDIO_LIST_ID}/{au['id']} - {au['title']}.m4a"
        print(f"下载 {au['title']}")
        resp: requests.Response = await sess.get(
            url,
            headers={
                "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0",
                "Referer": "https://www.bilibili.com/",
            },
            stream=True,
        )
        with open(file, "wb") as f:
            async for chunk in resp.aiter_content():
                if not chunk:
                    break
                f.write(chunk)


sync(main())
```
