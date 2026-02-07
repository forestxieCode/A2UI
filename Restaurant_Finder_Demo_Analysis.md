# 餐厅查找演示 (Restaurant Finder Demo) 分析与增强指南

本文档总结了对 `restaurant_finder` 演示中存在的限制进行的分析，并提供了启用动态数据生成的解决方案。

## 1. 当前行为分析

餐厅查找演示目前被设计为一个**静态概念验证 (PoC)**。目前，它只能成功处理符合特定标准的查询（例如：“Top 5 Chinese restaurants in New York”）。

### 为什么其他查询会失败？

如果你尝试 `Top 5 Italian restaurants in Rome`，系统将无法渲染 UI。这是由三个因素的连锁反应造成的：

1.  **工具中的严格地点过滤器**:
    `tools.py` 中的 `get_restaurants` 工具包含硬编码的检查：
    ```python
    if "new york" in location.lower() or "ny" in location.lower():
        # ... 读取文件的逻辑 ...
    ```
    任何非纽约的地点都会导致返回空列表 `[]`。

2.  **有限的静态数据**:
    演示依赖于 `restaurant_data.json`，其中仅包含 **8 条固定的** 位于纽约的中式/亚洲餐厅记录。没有实时搜索能力。

3.  **LLM 与 协议的严格性**:
    当工具不返回数据时，LLM (Gemini) 会智能地决定 *不* 生成 UI（因为没有内容可展示），而是生成一段文本道歉（“Sorry, I couldn't find...”）。
    然而，Agent 代码 (`agent.py`) 强制执行严格的验证规则：
    ```python
    if "---a2ui_JSON---" not in final_response_content:
        raise ValueError("Delimiter '---a2ui_JSON---' not found.")
    ```
    这导致纯文本道歉被视为**系统错误**，最终导致客户端崩溃或显示通用失败消息。

## 2. 系统工作流逻辑

```mermaid
graph TD
    User[用户输入] --> Client[Shell 客户端]
    Client --> Agent[Python Agent]
    Agent --> LLM[Gemini 模型]
    LLM -- 1. 调用工具 --> Tool[get_restaurants]
    
    Tool -- "查询: NY, Chinese" --> DataMatch[加载 restaurant_data.json]
    DataMatch --> LLM
    
    Tool -- "查询: Rome, Italian" --> NoMatch[返回空列表 []]
    NoMatch --> LLM
    
    LLM -- "有数据" --> GenUI[生成 A2UI JSON]
    GenUI --> Client
    
    LLM -- "无数据" --> GenText[生成文本道歉]
    GenText --> Error[Agent 验证错误]
```

## 3. 解决方案：动态模拟数据生成

为了使演示更加健壮并能够处理 *任何* 查询（例如，“Italian in Rome”，“Spicy in Chengdu”），我们可以修改 `get_restaurants` 工具，以便在静态文件不适用时生成**合成模拟数据**。

### 实现步骤

修改 `samples/agent/adk/restaurant_finder/tools.py`:

1.  导入 `random` 库。
2.  添加通用图像文件名列表。
3.  添加逻辑：如果静态数据查找失败或为空，则生成随机餐厅数据。

### 代码片段

```python
import json
import logging
import os
import random  # 新增

from google.adk.tools.tool_context import ToolContext

logger = logging.getLogger(__name__)

# 通用图片，确保 UI 显示正常
IMAGES = [
    "shrimpchowmein.jpeg", "mapotofu.jpeg", "beefbroccoli.jpeg", 
    "springrolls.jpeg", "kungpao.jpeg", "sweetsourpork.jpeg",
    "vegfriedrice.jpeg", "lasagna.png"
]

def get_restaurants(cuisine: str, location: str,  tool_context: ToolContext, count: int = 5) -> str:
    logger.info(f"--- TOOL CALLED: get_restaurants (count: {count}) ---")
    logger.info(f"  - Cuisine: {cuisine}")
    logger.info(f"  - Location: {location}")

    items = []
    base_url = tool_context.state.get("base_url", "http://localhost:10002")

    # 1. 静态数据策略 (保留纽约的现有逻辑)
    if "new york" in location.lower() or "ny" in location.lower():
        try:
            # ... 现有的文件读取逻辑 ...
            # (详情见原文件)
            pass 
        except Exception as e:
            logger.error(f"Error loading static data: {e}")

    # 2. 动态回退策略
    # 如果静态数据失败或地点不是 NY，则生成模拟数据
    if not items:
        logger.info("  - Generating mock data for arbitrary request.")
        for i in range(count):
            img_file = random.choice(IMAGES)
            stars = random.randint(3, 5)
            
            items.append({
                "name": f"{cuisine.title()} Delight #{i+1}",
                "detail": f"Authentic {cuisine} cuisine serving the best dishes in {location}.",
                "imageUrl": f"{base_url}/static/{img_file}",
                "rating": "★" * stars + "☆" * (5 - stars),
                "infoLink": f"[More Info](https://example.com/restaurant_{i})",
                "address": f"{random.randint(100, 999)} {location} Ave, {location}"
            })

    return json.dumps(items)
```

## 4. 总结

通过实现此回退机制，Agent 将始终向 LLM 返回数据。随后 LLM 将始终能够生成有效的 `A2UI` JSON 响应，从而满足 `agent.py` 中的严格验证，并防止“渲染失败 (Rendering failed)”错误。
