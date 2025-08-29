# AWS Support Case Agent with MCP Integration

Amazon Bedrock AgentCore 可帮助您安全、大规模地部署和运行功能强大的人工智能代理。它提供专为动态代理工作负载构建的基础设施、可增强代理功能的强大工具，以及适用于现实部署场景的基础控件。AgentCore 服务可以组合使用，也可以单独使用。该服务可兼容任何框架（包括 CrewAI、LangGraph、LlamaIndex 和 Strands Agents 等），并支持 Amazon Bedrock 内外的所有基础模型，能为您带来极大的灵活性。AgentCore 可消除构建专用代理基础设施时千篇一律的繁重工作，让您可以加快代理进入量产的过程。

当前大部分企业已经开发出对应的MCP Server，虽然AWS Gateway目前支持通过通过Lambda，OpenAPI schemas，REST AIP schemas来构建tool，但显然将企业现有的MCP Server源代码直接托管到Agentcore上能极大的节省改造成本。后续的主要工作则是如何构建Agent从而完成对托管在AgentCore Runtime上的MCP的调用。
这里以智能管理AWS Support Case为场景，介绍如果通过Agent调用托管的MCP Server，从而实现AgentCore Runtime（Agent）到AgentCore Runtime（MCP）的调用。该方案已经包含了流式处理、认证授权、提示词优化等，从而帮你轻松的将自建的Agent+MCP业务移植到AWS AgentCore上。

## 🚀 功能特性

- **智能支持案例管理**: 自动化AWS支持案例创建、跟踪和解决
- **智能支持案例总结和洞察**: 支持只能对AWS Case进行总结分析并给出最佳实践和洞察
- **MCP集成**: 利用模型上下文协议增强工具能力
- **跨境电商专用**: 专为国际业务IT支持场景优化
- **Bedrock Agent Core**: 基于AWS Bedrock Agent Core构建，支持可扩展的AI交互
- **可将现有MCP Server做无缝移植到AgentCore Runtime上**: 支持北京时间(UTC+8)的实时支持案例处理
- **构建Agent调用MCP Server**: 在AgentCore Runtime上额外构建Agent，从而实现智能调用MCP Tools

## 📁 项目结构

```
support-agent/
├── MCP/                           # MCP实现
│   ├── utils.py                   # 工具函数
│   ├── MCP_On_AgentCore.ipynb     # MCP notebook界面
│   ├── requirements.txt           # MCP依赖
│   └── awslabs/                   # AWS Labs MCP服务器
│       └── aws_support_mcp_server/ # 支持专用MCP服务器
└── README.md                      # 本文件
├── Agent/                          # Bedrock Agent实现
│   ├── Case_Agent_On_AgentCore.ipynb   # Jupyter notebook界面
│   ├── requirements.txt           # Agent依赖
│   └── Dockerfile                 # Agent的Dockerfile
```

## 🏗️ 系统架构

<img width="813" height="422" alt="iShot_2025-08-29_01 08 37" src="https://github.com/user-attachments/assets/1ea5b964-68fb-40de-ba22-4c72befec92b" />


## 🛠️ 前置条件

### AWS服务要求
- Python 3.10+
- Jupyter Notebook或JupyterLab
- AWS CLI已配置适当权限
- 开通AWS Support API访问权限(商业或企业支持计划)

### ⚠️ 重要：Bedrock模型访问
在开始之前，**必须**在AWS Bedrock控制台中开通所需的模型访问权限：

1. 登录AWS控制台，进入Amazon Bedrock服务
2. 在左侧导航栏选择"Model access"(模型访问)
3. 点击"Modify model access"(管理模型访问)
4. 选择并启用以下推荐模型：
   - **Claude 3.7 Sonnet** (推荐用于复杂推理)
   - **Claude 3 Haiku** (推荐用于快速响应)
   - 或其他支持的Anthropic Claude模型
5. 提交访问请求并等待批准(通常几分钟内完成)

**注意**: 没有模型访问权限，Agent将无法正常工作。

## 📦 安装步骤

### 推荐安装顺序：先MCP，后Agent

#### 第一步：MCP设置
1. 克隆仓库到本地
2. 打开 `MCP/MCP_On_AgentCore.ipynb`
3. **按顺序执行所有notebook单元格** - notebook包含所有必要的安装和配置步骤
4. 在notebook中可验证MCP服务器正常运行

#### 第二步：Agent设置  
1. 打开 `Agent/Case_Agent_On_AgentCore.ipynb`
2. **按顺序执行所有notebook单元格**
3. 等待Agent部署完成(可能需要几分钟)
4. 在notebook中可验证调用Agent的效果

## 🚀 使用方法

### 可参考如下代码编写调用Agent的代码

Case_Agent_On_AgentCore.ipynb notebook中有内置客户端调用Agent代码，可以直接使用：

```Python
import boto3
import json
import codecs
import argparse
from IPython.display import Markdown, display
from boto3.session import Session

prompt_text = "请分析我过去半年的case并给出最佳实践建议和洞察"

def invoke_agent(prompt_text):
    boto_session = Session()
    REGION = boto_session.region_name

    agent_arn = launch_result.agent_arn  # ⚠️ 确保 launch_result 已经定义
    # agent_arn='arn:aws:bedrock-agentcore:us-west-2:xxxxxxxxxxxx:runtime/agentcore_agent_invoke_agentcore_mcp_test-7OoavJDoxG'

    agentcore_client = boto3.client("bedrock-agentcore", region_name=REGION)

    boto3_response = agentcore_client.invoke_agent_runtime(
        agentRuntimeArn=agent_arn,
        qualifier="DEFAULT",
        payload=json.dumps({"prompt": prompt_text})
    )

    print(f"boto3_response: {boto3_response}")

    # ---- 处理流式响应 ----
    if "text/event-stream" in boto3_response.get("contentType", ""):
        print("Processing streaming response with boto3:")
        content = []
        for line in boto3_response["response"].iter_lines(chunk_size=10):
            if line:
                line = line.decode("utf-8")
                if line.startswith("data: "):
                    data = line[6:].replace('"', "")  # Remove "data: " prefix
                    data = data.replace("\\n", "\n")
                    print(f"{data}", end="")
                    content.append(data.replace('"', ""))
        # Display the complete streamed response
        full_response = " ".join(content)
        display(Markdown(full_response))
    else:
        try:
            events = []
            for event in boto3_response.get("response", []):
                events.append(event)
        except Exception as e:
            events = [f"Error reading EventStream: {e}"]

        if events:
            try:
                response_data = json.loads(events[0].decode("utf-8"))
                display(Markdown(response_data))
            except:
                print(f"Raw response: {events[0]}")


if __name__ == "__main__":
    print(f"Using prompt: {prompt_text}")
    invoke_agent(prompt_text)
```

## 🔒 安全考虑

- 使用AWS IAM角色和策略进行安全访问
- 凭证存储在AWS Secrets Manager中
- 支持VPC端点进行私有连接
- 所有支持案例操作的审计日志

## 🆘 故障排除

### 常见问题

1. **模型访问错误**: 确保在Bedrock控制台中启用了模型访问
2. **权限错误**: 检查AWS凭证和IAM权限
3. **部署失败**: 查看CloudWatch日志获取详细错误信息
4. **MCP连接问题**: 验证MCP服务器状态和网络连接

### 日志位置

- Agent运行时日志Cloudwatch log: `/aws/bedrock-agentcore/runtimes/{Agent Runtime ID}-DEFAULT`
- CodeBuild日志CloudWatch log: `/aws/codebuild/bedrock-agentcore-{Agent Name}-builder`

## 🤝 贡献

1. Fork仓库
2. 创建功能分支
3. 进行更改
4. 如适用，添加测试
5. 提交Pull Request

## 📄 许可证

本项目基于Apache License 2.0许可证 - 详见LICENSE文件。

## 🆘 支持

如有问题和疑问：
1. 查看现有GitHub issues
2. 创建新issue并提供详细描述
3. 对于AWS特定问题，请参考AWS Support文档

## 🔄 版本历史

- **v1.0.0**: 初始版本，基本Agent和MCP集成
- **当前版本**: 增强的跨境电商支持功能

---

**注意**: 本项目需要AWS商业或企业支持计划才能完全访问AWS Support API。
