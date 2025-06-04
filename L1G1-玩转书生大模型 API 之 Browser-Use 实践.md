# L1G1-玩转书生大模型 API 之 Browser-Use 实践

1.  构建一个包含 3 轮对话的上下文，调用 API 获取最终回复 and 发送一张图片并获取描述。
 2.  自行探索，并使用 Browser-use Web-UI 自带的录制功能，将书生大模型操作浏览器过程录制下来，放到作业中。

## 任务一
借助openai库完成对多轮问话的构建，指导文章中使用文件交互方法来读取API key，为了简便，笔者直接将API key存入字符串变量中（这种方式不安全），文章中使用
```
touch .env
```
创建环境变量文件，并用os库加载，原文中给出了单轮调用的代码示例（基于openai库）：
```
from openai import OpenAI  
from dotenv import load_dotenv  
import os  
  
InternLM_api_key = os.getenv("InternLM", load_dotenv())  
client = OpenAI(  
    api_key=InternLM_api_key,    
base_url="https://chat.intern-ai.org.cn/api/v1/",  
)  
  
chat_rsp = client.chat.completions.create(  
     model="internlm3-latest",  
     messages=[{  
            "role": "user",         #role 支持 user/assistant/system/tool            "content": "你知道刘慈欣吗？"  
    }, {  
            "role": "assistant",  
            "content": "为一个人工智能助手，我知道刘慈欣。他是一位著名的中国科幻小说家和工程师，曾经获得过多项奖项，包括雨果奖、星云奖等。"  
    },{  
            "role": "user",  
            "content": "他什么作品得过雨果奖？"  
    }],  
    stream=False  
)  
  
for choice in chat_rsp.choices:  
    print(choice.message.content)  
#若使用流式调用：stream=True，则使用下面这段代码  
#for chunk in chat_rsp:  
#    print(chunk.choices[0].delta.content)
```
以及不使用openai库的原生调用方法：
```
import requests  
import json  
from dotenv import load_dotenv  
import os  
InternLM_api_key = os.getenv("InternLM", load_dotenv())  
  
url = 'https://chat.intern-ai.org.cn/api/v1/chat/completions'  
header = {  
    'Content-Type':'application/json',  
    "Authorization":"Bearer "+InternLM_api_key,  
}  
data = {  
    "model": "internlm3-latest",    
	"messages": [{  
        "role": "user",  
        "content": "你好~"  
    }],  
    "n": 1,  
    "temperature": 0.8,  
    "top_p": 0.9  
}  
  
res = requests.post(url, headers=header, data=json.dumps(data))  
print(res.status_code)  
print(res.json())  
print(res.json()["choices"][0]['message']["content"])
```
简单介绍参数"role"的作用：

 - `role`：表示用户输入的内容，必须包含在请求的对话历史中，**通常是最后一条消息**；可包含问题、指令或对话上下文（如多轮对话中的用户发言）。
 - `assistant`：记录模型的历史回复，帮助模型理解自身之前的回答逻辑，保持多轮对话的一致性。仅在多轮对话中需要传递历史记录时使用；**需与`user`角色交替出现**，形成对话的“一问一答”结构。
 - `system`：提供全局性指导，设定模型的角色、回答风格或限制条件。通常放在对话列表的开头，且一条`system`消息即可生效；不是所有模型都支持`system`角色。

文章中同样给出了多模态调用方法的示例，更改content格式即可做到：
```
"content": [                                    #用户的图文提问内容，数组形式  
                {  
                    "type": "text",                        # 支持 text/image_url                    "text": "Describe these two images please"  
                },  
                {  
                    "type": "image_url",  
                    "image_url": {  
                        "url": "https://static.openxlab.org.cn/internvl/demo/visionpro.png"  #支持互联网公开可访问的图片 url 或图片的 base64 编码  
                    }  
                },  
                {  
                    "type": "image_url",                                                     # 单轮对话支持上传多张图片  
                    "image_url": {  
                        "url": "https://static.openxlab.org.cn/puyu/demo/000-2x.jpg"  
                    }  
                }  
            ]
```
为了实现多轮对话，我们将user信息和assistance信息存储起来：
```
from openai import OpenAI  
from dotenv import load_dotenv  
  
InternLM_api_key = "API_key"  
client = OpenAI(  
    api_key=InternLM_api_key,  
    base_url="https://chat.intern-ai.org.cn/api/v1/"  
)  
messages = []  
for i in range(3):  
    messages.append({  
        "role": "user",  
        "content": input()  
    })  
  
    chat_response = client.chat.completions.create(  
        model="internlm3-latest",  
        messages=messages,  
        n=1,  
        stream=False  
    )  
  
    for choices in chat_response.choices:  
        print(choices.message.content)  
    messages.append({  
        "role": "assistant",  
        "content": chat_response.choices[0].message.content  
    })
```
构建的对话过程如下：
![](/imgs/L1G1-1.1.jpg)
![](/imgs/L1G1-1.2.jpg)

## 任务二
Browser-Use 是一款专为 Agent 与浏览器交互设计的工具，旨在通过简单而强大的自动化界面，让 Agent 轻松访问和操作网页。它提供了连接大模型与浏览器的便捷桥梁，使开发者能够快速实现网页自动化任务，无需复杂编码。
原文中通过uv进行环境管理，但是因为笔者本地已经有anaconda，所以并未使用uv，笔者为了节省C盘空间，在D盘下新建了虚拟环境仓库
 ```
conda creat --prefix D:\anaconda_envs\internlm python=3.11
```
注意python版本应该在3.11以上。删除此环境使用
```
conda env remove --prefix D:\anaconda_envs\internlm
```
进入环境
```
conda activate D:\anaconda_envs\internlm
```
进入项目路径
```
cd web-ui
```
安装依赖
```
pip install -r requirements.txt
```
启动时出现错误
```
ERROR:    Exception in ASGI application  
…………  
TypeError: argument of type 'bool' is not iterable
```
更新相关包以解决
```
pip install --upgrade gradio  -i https://pypi.tuna.tsinghua.edu.cn/simple/
```
运行WebUI
```
python webui.py --ip 127.0.0.1 --port 7788
```
在搜索问题的时候出现
```
ERROR    [browser] Failed to initialize Playwright browser: BrowserType.launch: Executable doesn't exist at C:\Users\...\chromium-1169\chrome-win\chrome.exe
```
重装Playwright
```
playwright install
```
重新搜索问题发现能够实现Agent与浏览器的互动
![输入图片说明](/imgs/L1G1-2.1.gif)


> Written with [StackEdit中文版](https://stackedit.cn/).
