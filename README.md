
### 本项目fork自用户@ Jagroop2001开发的【chat-with-pdf】这个project！

我们将构建一个简单的聊天界面，允许用户上传PDF文件，使用OpenAI的API检索其内容，并在类似聊天的界面中显示响应。我们还将使用@pinata上传和存储PDF文件。

## 我们即将构建的项目预览：

### 前提条件：

- 基本的Python知识
- Pinata API密钥（用于上传PDF）
- OpenAI API密钥（用于生成响应）
- 安装了Streamlit（用于构建用户界面）

### 第1步：项目设置

首先创建一个新的Python项目目录：

```bash
mkdir chat-with-pdf
cd chat-with-pdf
python3 -m venv venv
source venv/bin/activate
pip install streamlit openai requests PyPDF2
```

接着，在项目的根目录创建一个`.env`文件，并添加以下环境变量：

```bash
PINATA_API_KEY=<你的Pinata API密钥>
PINATA_SECRET_API_KEY=<你的Pinata密钥>
OPENAI_API_KEY=<你的OpenAI API密钥>
```

你需要自己管理OPENAI_API_KEY，因为它是付费的。接下来，我们将讲解如何在Pinata中创建API密钥。

### 皮纳塔（Pinata）是什么？

Pinata是一个为IPFS（星际文件系统）提供文件存储和管理的平台，是一种去中心化的分布式文件存储系统。

- **去中心化存储**：Pinata帮助你在IPFS上存储文件，这是一个去中心化的网络。
- **易于使用**：它提供了用户友好的工具和API进行文件管理。
- **文件可用性**：Pinata通过将文件固定在IPFS上确保它们的可访问性。
- **NFT支持**：非常适合存储NFT和Web3应用的元数据。
- **经济实惠**：相比传统云存储，Pinata可能是一种更具性价比的选择。

### 第2步：使用Pinata上传PDF

我们将使用Pinata的API来上传PDF文件，并获取每个文件的哈希值（CID）。创建一个名为`pinata_helper.py`的文件，用于处理PDF上传。

```python
import os
import requests
from dotenv import load_dotenv

# 加载环境变量
load_dotenv()

# Pinata API URL，用于将文件固定到IPFS
PINATA_API_URL = "https://api.pinata.cloud/pinning/pinFileToIPFS"

# 从环境变量中获取Pinata API密钥
PINATA_API_KEY = os.getenv("PINATA_API_KEY")
PINATA_SECRET_API_KEY = os.getenv("PINATA_SECRET_API_KEY")

def upload_pdf_to_pinata(file_path):
    headers = {
        "pinata_api_key": PINATA_API_KEY,
        "pinata_secret_api_key": PINATA_SECRET_API_KEY
    }

    with open(file_path, 'rb') as file:
        response = requests.post(PINATA_API_URL, files={'file': file}, headers=headers)

        if response.status_code == 200:
            print("文件上传成功")
            return response.json()['IpfsHash']
        else:
            print(f"错误: {response.text}")
            return None
```

### 第3步：设置OpenAI

接下来，我们将创建一个函数，使用OpenAI的API与从PDF提取的文本进行交互。创建一个名为`openai_helper.py`的文件：

```python
import os
from openai import OpenAI
from dotenv import load_dotenv

load_dotenv()

OPENAI_API_KEY = os.getenv("OPENAI_API_KEY")
client = OpenAI(api_key=OPENAI_API_KEY)

def get_openai_response(text, pdf_text):
    try:
        messages = [
            {"role": "system", "content": "你是一名帮助回答PDF问题的助手。"},
            {"role": "user", "content": pdf_text},
            {"role": "user", "content": text}
        ]

        response = client.chat.completions.create(
            model="gpt-4",
            messages=messages,
            max_tokens=100,
            temperature=0.7
        )

        return response.choices[0].message.content
    except Exception as e:
        return f"错误: {str(e)}"
```

### 第4步：构建Streamlit界面

现在，我们已经准备好辅助函数，接下来构建一个Streamlit应用，用于上传PDF文件、从OpenAI获取响应并显示聊天。

创建一个名为`app.py`的文件：

```python
import streamlit as st
import os
import time
from pinata_helper import upload_pdf_to_pinata
from openai_helper import get_openai_response
from PyPDF2 import PdfReader
from dotenv import load_dotenv

# 加载环境变量
load_dotenv()

st.set_page_config(page_title="与PDF聊天", layout="centered")

st.title("使用OpenAI和Pinata与PDF聊天")

uploaded_file = st.file_uploader("上传你的PDF文件", type="pdf")

if "chat_history" not in st.session_state:
    st.session_state.chat_history = []
if "loading" not in st.session_state:
    st.session_state.loading = False

if uploaded_file is not None:
    file_path = os.path.join("temp", uploaded_file.name)
    with open(file_path, "wb") as f:
        f.write(uploaded_file.getbuffer())

    st.write("正在将PDF上传到Pinata...")
    pdf_cid = upload_pdf_to_pinata(file_path)

    if pdf_cid:
        st.write(f"文件已上传到IPFS，CID为: {pdf_cid}")

        reader = PdfReader(file_path)
        pdf_text = ""
        for page in reader.pages:
            pdf_text += page.extract_text()

        if pdf_text:
            st.text_area("PDF内容", pdf_text, height=200)

            user_input = st.text_input("向PDF提问：", disabled=st.session_state.loading)

            if st.button("发送", disabled=st.session_state.loading):
                if user_input:
                    st.session_state.loading = True

                    with st.spinner("AI思考中..."):
                        time.sleep(1)
                        response = get_openai_response(user_input, pdf_text)

                    st.session_state.chat_history.append({"user": user_input, "ai": response})

                    st.session_state.input_text = ""
                    st.session_state.loading = False

            if st.session_state.chat_history:
                for chat in st.session_state.chat_history:
                    st.write(f"**你:** {chat['user']}")
                    st.write(f"**AI:** {chat['ai']}")
        else:
            st.error("无法从PDF中提取文本。")
    else:
        st.error("PDF上传到Pinata失败。")
```

### 第5步：运行应用

使用以下命令在本地运行应用：

```bash
streamlit run app.py
```

### 第6步：代码解析

- **Pinata上传**：用户上传PDF文件后，临时保存到本地，并通过`upload_pdf_to_pinata`函数上传到Pinata，Pinata返回一个表示文件在IPFS上的哈希值（CID）。
- **PDF提取**：上传成功后，使用PyPDF2提取PDF内容并显示在文本区域。
- **OpenAI交互**：用户可以通过输入框向PDF内容提问，`get_openai_response`函数将用户问题和PDF内容发送到OpenAI，并返回相关的响应。
