+++
title = 'Qwen命令行小脚本(破事水)'
date = 2024-07-11T22:02:56+08:00
draft = false
+++

# 使用 Python 实现 Qwen CLI 交互

## 我为啥要整这玩意?

没啥就闲得慌, 最近在补 NJU jyy OS 课程, 老师上课演示有一个 ag(askGPT) 命令(应该是自己写的)调用GPT API. 闲着无聊想整一个类似的, 刚好通义千问的 API 很便宜. 新人注册阿里云灵积会给每个模型赠送限时一个月的 1e6 token(对qwen-long qwen-turbo qwen-max qwen-plus 等模型分别赠送一百万token)

## 开整

```python
import argparse
import requests
import os
import re


API_URL = (
    "https://dashscope.aliyuncs.com/api/v1/services/aigc/text-generation/generation"
)
API_KEY = os.getenv("DASHSCOPE_API_KEY")
# for streaming output
headers = {
    "Content-Type": "application/json",
    "Authorization": f"Bearer {API_KEY}",
    "X-DashScope-SSE": "enable",
}
pattern = re.compile(r'"content":"(.*?)","role"')
model_name_list = ["qwen-long", "qwen-turbo", "qwen-plus", "qwen-max"]


def strProcess(res_text):
    # replace multiple literal '\n' with one '\n'
    res_text = re.sub(r'(\\n)+', '\n', res_text)
    # replace \" with "
    res_text = res_text.replace('\\"', '"')
    # replace \' with '
    res_text = res_text.replace("\\'", "'")
    return res_text


def chat(model_name="qwen-turbo"):
    assert model_name in model_name_list, f"model_name should be in {model_name_list}"
    messages = [{"role": "system", "content": "You are a helpful assistant."}]
    round_num = 0
    print(f"Welcome to {model_name}! You can type 'e/q' to quit.")
    while True:
        round_num += 1
        user_input = input(f"Q{round_num}: ")
        exit_flag = ["exit", "quit", "q", "e"]
        if user_input.lower() in exit_flag:
            exit(0)
        messages.append({"role": "user", "content": user_input})
        body = {
            "model": model_name,
            "input": {
                "messages": messages,
            },
            "parameters": {
                "result_format": "message",
                # streaming output
                "incremental_output": True,
            },
        }
        response = requests.post(API_URL, headers=headers, json=body, stream=True)
        # response = requests.post(API_URL, headers=headers, json=body, stream=True, proxies=None)

        response_content = ""
        print(f"A{round_num}: ", end="")
        for chunk in response.iter_content(chunk_size=None):
            chunk = chunk.decode("utf-8")
            match = pattern.search(chunk)
            if match:
                # Catch the content in the response
                print_text = match.group(1)
                response_content += print_text
                # Process the content to make it more readable
                print_text = strProcess(print_text)
                print(print_text, end="", flush=True)

        messages.append({"role": "assistant", "content": response_content})
        print("\n")


if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument(
        "--n",
        type=str,
        default="turbo",
        help=f"model_name should be in {model_name_list}",
    )
    args = parser.parse_args()
    model_name = "qwen-turbo"
    if args.n:
        model_name = "qwen" + "-" + args.n
    chat(model_name)
```

然后到 `.bashrc` 或者 `.zshrc` 加个别名

```sh
alias qwen="python3 /path/to/qwen.py"
```

命令行也可以加参数 `--n max` 用来调用 `qwen-max` 模型, 默认为 `qwen-turbo`

这个脚本实际上没啥难的(所以说是破事水). 但如果正常发包的话, 需要等模型完全输出答案之后本地才能突然 print 出全部输出, 这等起来实在讨厌. 所以加一个流式传输, 就能像浏览器里面一样, 一个字一个字地蹦出回答了. 然后再把流式输出聚成一坨加入 messages 实现多轮对话

## 可能还有些问题?

实际上我在日常使用的时候, Qwen 有时候会长时间卡住, 不清楚是我设置了代理导致的还是阿里云服务器负载太高. 我尝试过设置 POST `proxies=None`, 也有在调用前 unset 所有代理, 不过不管用
