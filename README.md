# 🤖 AutoGen Multi-Agent Chat System

<div align="center">

**Multi-Agent AI Assistant System with AutoGen, Panel, and OpenAI**

[![AutoGen](https://img.shields.io/badge/AutoGen-%2300A67E.svg?style=for-the-badge&logo=microsoft&logoColor=white)](https://microsoft.github.io/autogen/)
[![OpenAI](https://img.shields.io/badge/OpenAI-412991.svg?style=for-the-badge&logo=openai&logoColor=white)](https://openai.com/)
[![Panel](https://img.shields.io/badge/Panel-%2300A67E.svg?style=for-the-badge&logo=python&logoColor=white)](https://panel.holoviz.org/)

*Multi-agent AI system with specialized roles for complex task solving*

</div>

## 📋 Project Overview

This project implements a sophisticated **multi-agent AI system** using Microsoft's AutoGen framework. It features specialized agent teams for different tasks including application development, content creation, and data analysis, all coordinated through an interactive web interface built with Panel.

## 🎯 Key Features

- 🤝 **Multi-Agent Collaboration** - Teams of specialized AI agents working together
- 🎭 **Specialized Agent Roles** - Coder, Scientist, Planner, Critic, Content Creator, Reviewer
- 🌐 **Web Interface** - Interactive UI built with Panel
- 🔄 **Async Human Input** - Custom human-in-the-loop interaction
- 🏗️ **Modular Architecture** - Easily extensible agent groups
- 📊 **Group Chat Management** - Coordinated multi-agent conversations

## 🚀 Quick Start

### Installation

```bash
# Install core dependencies
pip install pyautogen
pip install panel
pip install openai

# Optional: for enhanced functionality
pip install asyncio
pip install nest-asyncio
```

### Environment Setup

```python
import os
from autogen import AssistantAgent, UserProxyAgent

# Set your OpenAI API key
os.environ["OPENAI_API_KEY"] = "your-api-key-here"

# LLM configuration
llm_config = {
    "config_list": [
        {
            "model": "gpt-4",
            "api_key": os.environ["OPENAI_API_KEY"],
        }
    ],
    "temperature": 0.7,
    "timeout": 120,
}
```

## 🏗️ System Architecture

### Core Components

```python
import autogen
import panel as pn
import asyncio
from chat_state import ChatState

# Global chat state management
chat_state = ChatState.getInstance()

class MyConversableAgent(autogen.ConversableAgent):
    """Custom agent with async human input handling"""
    async def a_get_human_input(self, prompt: str) -> str:
        chat_state.input_future
        print('AGET!!!!!!')  

        if chat_state.input_future is None or chat_state.input_future.done():
            chat_state.input_future = asyncio.Future()
        await chat_state.input_future
        input_value = chat_state.input_future.result()
        chat_state.input_future = None
        return input_value
```

## 👥 Agent Teams

### 1. Application Development Group

```python
class application_group:     
    def __init__(self, llm_config):
        self.llm_config = llm_config
        self.init_agents()

    def init_agents(self):
        # Admin Agent - Human proxy with approval authority
        self.user_proxy = MyConversableAgent(
            name="Admin",
            max_consecutive_auto_reply=20,
            is_termination_msg=lambda x: x.get("content", "").rstrip().endswith("TERMINATE"),
            system_message="""A human admin. Interact with the planner to discuss the plan. Plan execution needs to be approved by this admin. 
            Don't reply thank you, only reply TERMINATE if the task has been solved at full satisfaction.
            Otherwise, reply CONTINUE, or the reason why the task is not solved yet.
            """,
            code_execution_config={"work_dir": "web"},
            human_input_mode="ALWAYS",
            llm_config=self.llm_config,
        )

        # Coder Agent - Technical implementation specialist
        self.coder = autogen.AssistantAgent(
            name="Coder",
            human_input_mode="NEVER",
            llm_config=self.llm_config,
            system_message='''Engineer. You follow an approved plan. You write python/shell code to solve tasks. Wrap the code in a code block that specifies the script type. The user can't modify your code. So do not suggest incomplete code which requires others to modify. Don't use a code block if it's not intended to be executed by the executor.
            Don't include multiple code blocks in one response. Do not ask others to copy and paste the result. Check the execution result returned by the executor.
            If the result indicates there is an error, fix the error and output the code again. Suggest the full code instead of partial code or code changes.
            '''
        )

        # Scientist Agent - Research and analysis specialist
        self.scientist = autogen.AssistantAgent(
            name="Scientist",
            human_input_mode="NEVER",
            llm_config=self.llm_config,
            is_termination_msg=lambda x: x.get("content", "").rstrip().endswith("TERMINATE"),
            system_message="""Scientist. You follow an approved plan. If have paper, you are able to categorize papers after seeing their abstracts printed. You don't write code.
            Don't reply thank you, only reply TERMINATE if the task has been solved at full satisfaction.
            Otherwise, reply CONTINUE, or the reason why the task is not solved yet.
            """
        )

        # Planner Agent - Task decomposition and strategy
        self.planner = autogen.AssistantAgent(
            name="Planner",
            human_input_mode="NEVER",
            is_termination_msg=lambda x: x.get("content", "").rstrip().endswith("TERMINATE"),
            system_message='''Planner. Suggest a plan. Revise the plan based on feedback from admin and critic, until admin approval.
            The plan may involve an engineer who can write code and a scientist who doesn't write code.
            Explain the plan first. Be clear which step is performed by an engineer, and which step is performed by a scientist.
            '''
        )

        # Executor Agent - Code execution and validation
        self.executor = autogen.UserProxyAgent(
            name="Executor",
            system_message="""Executor. Execute the code written by the engineer and report the result. Reply TERMINATE if the task has been solved at full satisfaction.
            Otherwise, reply CONTINUE, or the reason why the task is not solved yet.""",
            human_input_mode="NEVER",
            code_execution_config={"work_dir": "web"},
        )

        # Critic Agent - Quality assurance and evaluation
        self.critic = autogen.AssistantAgent(
            name="Critic",
            system_message="""Critic. You are a helpful assistant highly skilled in evaluating the quality of a given visualization code by providing a score from 1 (bad) - 10 (good) while providing clear rationale. YOU MUST CONSIDER VISUALIZATION BEST PRACTICES for each evaluation. Specifically, you can carefully evaluate the code across the following dimensions
            - bugs (bugs):  are there bugs, logic errors, syntax error or typos?
            - Data transformation (transformation): Is the data transformed appropriately for the visualization type?
            - Goal compliance (compliance): how well the code meets the specified visualization goals?
            - Visualization type (type): Is the visualization type appropriate for the data and intent?
            - Data encoding (encoding): Is the data encoded appropriately for the visualization type?
            - aesthetics (aesthetics): Are the aesthetics of the visualization appropriate?
            """,
            is_termination_msg=lambda x: x.get("content", "").rstrip().endswith("TERMINATE"),
            llm_config= self.llm_config,
            human_input_mode="NEVER",
        )

        # Group Chat Coordination
        self.groupchat = autogen.GroupChat(
            agents=[self.user_proxy, self.coder, self.scientist, self.planner, self.executor, self.critic],
            messages=[],
            max_round=50,
        )
        self.manager = autogen.GroupChatManager(groupchat=self.groupchat, llm_config=self.llm_config)
```

### 2. Content Creation Group

```python
class content_group:     
    def __init__(self, llm_config):
        self.llm_config = llm_config
        self.init_agents()

    def init_agents(self):
        # Admin Agent - Task delegation and approval
        self.user_proxy = MyConversableAgent(
            name="Admin",
            is_termination_msg=lambda x: x.get("content", "") and x.get("content", "").rstrip().endswith("TERMINATE"),
            system_message="A human admin. You need to giving a task for all Employee.",
            code_execution_config=False,
            human_input_mode="ALWAYS",
            llm_config= self.llm_config,
        )

        # Content Creator Agent - Creative content generation
        self.content_creator = autogen.AssistantAgent(
            name="Content_creator",
            human_input_mode="NEVER",
            llm_config= self.llm_config,
            system_message="Content creator, you are an excellent content creator who can write about any topic requested by the admin. This includes a full range of diverse fields from content creation to writing. You will receive a content creation request from the admin and from there, please create content that corresponds with the request. After creating content, you should submit it to the Reviewer for evaluation and feedback. Only reply with TERMINATE once the entire project is completed.",
        )

        # Reviewer Agent - Content quality assessment
        self.reviewer = autogen.AssistantAgent(
            name="reviewer",
            human_input_mode="NEVER",
            llm_config= self.llm_config,
            system_message="As a reviewer, your key task is to assess and critique content from our creators. You'll evaluate its quality, relevance, and engagement, offering suggestions for improvement. Your responsibility to provide constructive feedback directly to the content creators. It's imperative that you consistently provide constructive feedback that encourages innovation and enhancement. Only reply with TERMINATE once the entire project is completed.",
        )

        # Content Group Chat
        self.groupchat = autogen.GroupChat(
            agents=[self.user_proxy, self.content_creator, self.reviewer],
            messages=[],
            max_round=20,
        )
        self.manager = autogen.GroupChatManager(groupchat=self.groupchat, llm_config=self.llm_config)
```

### 3. Simple Assistant Chat

```python
class assistant_chat:
    def __init__(self, llm_config):
        self.llm_config = llm_config
        self.init_agents()
        
    def init_agents(self):
        # Simple one-on-one assistant
        self.assistant = autogen.AssistantAgent(
            "Assistant",
            system_message='''You are a helpful assistant.''',
            llm_config=self.llm_config
        )
        
        self.user_proxy = MyConversableAgent(
            name="user",
            is_termination_msg=lambda x: x.get("content", "").rstrip().endswith("exit"),
            system_message="""A human admin.""",
            code_execution_config=False,
            human_input_mode="ALWAYS",
            llm_config=self.llm_config,
        )
```

## 🎯 Usage Examples

### Starting a Development Task

```python
# Initialize the application group
app_group = application_group(llm_config)

# Start a development task
app_group.user_proxy.initiate_chat(
    app_group.manager,
    message="Create a web application for data visualization using Plotly and Dash. The app should load CSV data and create interactive charts."
)
```

### Content Creation Workflow

```python
# Initialize content group
content_team = content_group(llm_config)

# Start content creation task
content_team.user_proxy.initiate_chat(
    content_team.manager,
    message="Write a comprehensive blog post about the benefits of multi-agent AI systems in enterprise applications. Include real-world use cases and implementation guidelines."
)
```

### Simple Assistant Interaction

```python
# Initialize simple assistant
assistant = assistant_chat(llm_config)

# Have a conversation
assistant.user_proxy.initiate_chat(
    assistant.assistant,
    message="Help me understand how machine learning models are deployed in production environments."
)
```

## 📊 Agent Roles & Responsibilities

| Agent | Primary Role | Key Responsibilities |
|-------|-------------|---------------------|
| **Admin** | Human Proxy | Task initiation, plan approval, termination control |
| **Planner** | Strategist | Task decomposition, workflow planning, resource allocation |
| **Coder** | Developer | Code implementation, debugging, technical solutions |
| **Scientist** | Analyst | Research, data analysis, paper review, non-coding tasks |
| **Executor** | Validator | Code execution, result verification, testing |
| **Critic** | QA Specialist | Code review, best practices, quality scoring |
| **Content Creator** | Writer | Content generation, creative writing, documentation |
| **Reviewer** | Editor | Content evaluation, feedback, quality assurance |

## 🔧 Configuration

### LLM Configuration

```python
# Advanced LLM configuration
advanced_llm_config = {
    "config_list": [
        {
            "model": "gpt-4",
            "api_key": os.environ["OPENAI_API_KEY"],
            "api_type": "openai",
            "api_base": "https://api.openai.com/v1",
        }
    ],
    "temperature": 0.7,
    "timeout": 120,
    "max_tokens": 4000,
    "seed": 42,
}

# Azure OpenAI configuration
azure_llm_config = {
    "config_list": [
        {
            "model": "gpt-4",
            "api_key": "your-azure-openai-api-key",
            "api_base": "https://your-resource.openai.azure.com/",
            "api_type": "azure",
            "api_version": "2023-08-01-preview"
        }
    ],
    "temperature": 0.7,
}
```

### Chat State Management

```python
# chat_state.py (example implementation)
import asyncio

class ChatState:
    _instance = None
    
    @classmethod
    def getInstance(cls):
        if cls._instance is None:
            cls._instance = ChatState()
        return cls._instance
    
    def __init__(self):
        self.input_future = None
        self.chat_history = []
        self.active_chats = {}
```

## 🌐 Web Interface Integration

### Panel Dashboard Setup

```python
import panel as pn

class ChatInterface:
    def __init__(self):
        pn.extension()
        self.setup_ui()
    
    def setup_ui(self):
        # Create chat interface components
        self.chat_window = pn.widgets.ChatBox(
            name="Multi-Agent Chat",
            placeholder="Type your message here..."
        )
        
        self.agent_selector = pn.widgets.Select(
            name="Select Agent Group",
            options=['Application Team', 'Content Team', 'Simple Assistant'],
            value='Application Team'
        )
        
        self.send_button = pn.widgets.Button(name="Send", button_type="primary")
        self.send_button.on_click(self.send_message)
        
        # Layout the dashboard
        self.dashboard = pn.Column(
            pn.Row("## 🤖 Multi-Agent Chat System", pn.layout.HSpacer()),
            pn.Row(self.agent_selector),
            self.chat_window,
            pn.Row(self.send_button)
        )
    
    def send_message(self, event):
        message = self.chat_window.value
        agent_group = self.agent_selector.value
        
        # Route message to appropriate agent group
        if agent_group == 'Application Team':
            self.send_to_application_team(message)
        elif agent_group == 'Content Team':
            self.send_to_content_team(message)
        else:
            self.send_to_assistant(message)

# Launch the interface
# chat_interface = ChatInterface()
# chat_interface.dashboard.servable()
```

## 🚀 Deployment

### Local Development

```bash
# Run the Panel application
panel serve main.py --show --autoreload

# Or run with specific parameters
panel serve main.py --port 5006 --show
```

### Production Deployment

```python
# For production deployment
if __name__ == "__main__":
    # Initialize all agent groups
    llm_config = {...}  # Your LLM configuration
    
    app_team = application_group(llm_config)
    content_team = content_group(llm_config)
    assistant = assistant_chat(llm_config)
    
    # Serve the Panel app
    pn.serve(
        {'app': create_dashboard(app_team, content_team, assistant)},
        port=5006,
        address='0.0.0.0',
        show=False
    )
```

## 📈 Use Cases

### Enterprise Applications
- **Software Development** - Multi-agent code review and implementation
- **Content Marketing** - Collaborative content creation and editing
- **Data Analysis** - Coordinated data processing and visualization
- **Research Assistance** - Literature review and scientific analysis

### Educational Applications
- **Tutoring Systems** - Multi-perspective learning assistance
- **Research Projects** - Collaborative academic writing and analysis
- **Code Education** - Step-by-step programming guidance

## ⚡ Performance Optimization

### Caching and State Management

```python
from functools import lru_cache

@lru_cache(maxsize=128)
def get_agent_group(group_type, llm_config):
    """Cache agent groups for better performance"""
    if group_type == "application":
        return application_group(llm_config)
    elif group_type == "content":
        return content_group(llm_config)
    else:
        return assistant_chat(llm_config)
```

## 🔒 Security Considerations

```python
# Secure API key management
import keyring

def get_secure_api_key(service_name="openai"):
    """Retrieve API key from secure storage"""
    return keyring.get_password("system", service_name)

# Input validation
def sanitize_user_input(user_input):
    """Basic input sanitization"""
    import html
    return html.escape(user_input)
```

---

<div align="center">

**Built with ❤️ using AutoGen, Panel, and OpenAI**

*Creating collaborative AI systems for complex problem-solving*

</div>
