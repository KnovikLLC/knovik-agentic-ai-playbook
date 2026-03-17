# Tools Integration

---

## What tools give agents

Without tools, a CrewAI agent is just an LLM that can reason and write. Tools give agents the ability to **act on the world** — search for current information, read and write files, call APIs, query databases.

Tools are assigned per-agent. An agent will only ever use the tools you give it. This is intentional — it keeps agent behavior predictable and auditable.

---

## Built-in CrewAI tools

CrewAI ships with a set of ready-made tools via the `crewai-tools` package:

```python
from crewai_tools import (
    SerperDevTool,       # Web search via Serper API
    FileReadTool,        # Read local files
    FileWriteTool,       # Write local files
    DirectoryReadTool,   # List and read a directory
    WebsiteSearchTool,   # Scrape and search a specific website
    PDFSearchTool,       # Search within PDF documents
    CSVSearchTool,       # Query CSV files
    YoutubeVideoSearchTool,  # Search YouTube video transcripts
)
```

**Web search setup:**
```python
import os
os.environ["SERPER_API_KEY"] = "your-serper-key"  # serper.dev — free tier available

search_tool = SerperDevTool()
```

---

## Custom tools — the Knovik way

For Knovik-specific integrations (Strapi CMS, WooCommerce, Shopify, internal APIs), you build custom tools using the `@tool` decorator. This is the pattern to use for all product integrations.

```python
from crewai import tool
import httpx

@tool("Fetch product data from ATHBridge WooCommerce")
def get_athbridge_products(category: str) -> str:
    """
    Fetches product listings from ATHBridge WooCommerce API for a given category.
    Returns product names, prices, ratings, and short descriptions.
    
    Args:
        category: Product category slug (e.g., 'zigbee-hubs', 'matter-hubs')
    """
    try:
        response = httpx.get(
            f"https://athbridge.com/wp-json/wc/v3/products",
            params={"category": category, "per_page": 10},
            auth=(
                os.getenv("WOOCOMMERCE_ATHBRIDGE_KEY"),
                os.getenv("WOOCOMMERCE_ATHBRIDGE_SECRET")
            ),
            timeout=10
        )
        products = response.json()
        result = []
        for p in products:
            result.append(
                f"- {p['name']}: ${p['price']} | Rating: {p.get('average_rating', 'N/A')} | {p['short_description'][:100]}"
            )
        return "\n".join(result) if result else "No products found in this category."
    except Exception as e:
        return f"Error fetching products: {str(e)}"
```

---

## Tool design principles

1. **Docstrings are system prompts for tool selection.**
   The agent reads the docstring to decide whether to call the tool. Write it as if explaining to a junior analyst when and why to use this tool.

2. **Always return strings.**
   CrewAI tools must return strings. If your function returns a dict or list, serialize it:
   ```python
   import json
   return json.dumps(result, indent=2)
   ```

3. **Handle errors gracefully — never raise.**
   If a tool raises an exception, the agent gets confused. Return an error string instead:
   ```python
   try:
       # ... tool logic
   except Exception as e:
       return f"Tool failed: {str(e)}. Try a different approach."
   ```

4. **Keep tool scope narrow.**
   One tool, one job. Do not build a `do_everything_marketing_tool`. Build `get_keyword_search_volume`, `get_competitor_keywords`, `get_serp_position` separately. Narrow tools are easier to debug and easier for agents to use correctly.

---

## Practical tool set for Knovik Marketing Platform

Here is the complete tool set used in the demo chapter:

```python
from crewai import tool
from crewai_tools import SerperDevTool
import httpx, os, json

# ── Search ──────────────────────────────────────────────────────────────────
search_tool = SerperDevTool()  # General web search

@tool("Search for keyword data")
def get_keyword_data(keyword: str) -> str:
    """
    Returns search volume, keyword difficulty, and CPC for a given keyword.
    Use this when you need SEO data for content planning.
    
    Args:
        keyword: The target keyword or phrase to analyse
    """
    # In production: call DataForSEO or Ahrefs API
    # Simulated for demo:
    return json.dumps({
        "keyword": keyword,
        "monthly_searches": 12000,
        "keyword_difficulty": 42,
        "cpc_usd": 1.20,
        "top_ranking_pages": [
            "athbridge.com/blog/best-zigbee-hubs",
            "addtohomekit.com/zigbee-hubs-guide",
            "reddit.com/r/homekit/zigbee-hubs"
        ]
    }, indent=2)


# ── Strapi CMS ───────────────────────────────────────────────────────────────
@tool("Fetch existing blog posts from Strapi CMS")
def get_existing_posts(tag: str) -> str:
    """
    Fetches existing blog posts from the Knovik Strapi CMS for a given tag.
    Use this to avoid writing duplicate content and to find internal linking opportunities.
    
    Args:
        tag: Content tag to search for (e.g., 'zigbee', 'homekit', 'matter-protocol')
    """
    try:
        response = httpx.get(
            f"{os.getenv('STRAPI_URL')}/api/articles",
            params={"filters[tags][name][$eq]": tag, "populate": "tags"},
            headers={"Authorization": f"Bearer {os.getenv('STRAPI_TOKEN')}"},
            timeout=10
        )
        articles = response.json().get("data", [])
        if not articles:
            return f"No existing posts found with tag: {tag}"
        result = [f"- {a['attributes']['title']} ({a['attributes']['slug']})" for a in articles]
        return f"Existing posts tagged '{tag}':\n" + "\n".join(result)
    except Exception as e:
        return f"CMS unavailable: {str(e)}"


# ── Ghost CMS Publisher ───────────────────────────────────────────────────────
@tool("Save draft to Ghost CMS")
def save_ghost_draft(title: str, content: str, tags: str) -> str:
    """
    Saves a blog post as a draft in Ghost CMS.
    Use this as the final step after content has been written and reviewed.
    
    Args:
        title: The post title
        content: Full post content in HTML or Markdown
        tags: Comma-separated tag names (e.g., 'zigbee,homekit,smart-home')
    """
    try:
        tag_list = [{"name": t.strip()} for t in tags.split(",")]
        payload = {
            "posts": [{
                "title": title,
                "html": content,
                "tags": tag_list,
                "status": "draft"
            }]
        }
        response = httpx.post(
            f"{os.getenv('GHOST_API_URL')}/ghost/api/admin/posts/",
            json=payload,
            headers={
                "Authorization": f"Ghost {os.getenv('GHOST_ADMIN_API_KEY')}",
                "Content-Type": "application/json"
            },
            timeout=15
        )
        post = response.json()["posts"][0]
        return f"Draft saved: {post['title']} — ID: {post['id']} — URL: {post['url']}"
    except Exception as e:
        return f"Failed to save draft: {str(e)}"
```

---

## Next

In {doc}`05_demo_content_crew` we wire all of this together into a complete, runnable Marketing Platform content crew.

