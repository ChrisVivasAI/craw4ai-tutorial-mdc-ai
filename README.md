# 🚀 Miami Dade College AI Student Guide: Web Scraping with Crawl4AI

<div align="center">

[![GitHub Stars](https://img.shields.io/github/stars/unclecode/crawl4ai?style=social)](https://github.com/unclecode/crawl4ai/stargazers)
[![Python Version](https://img.shields.io/pypi/pyversions/crawl4ai)](https://pypi.org/project/crawl4ai/)
[![License](https://img.shields.io/github/license/unclecode/crawl4ai)](https://github.com/unclecode/crawl4ai/blob/main/LICENSE)

</div>

Welcome, Miami Dade College AI students! This guide will help you set up and use [Crawl4AI](https://github.com/unclecode/crawl4ai), an open-source web crawler and scraper designed for AI applications. By the end of this guide, you'll have a working RAG (Retrieval-Augmented Generation) system that can answer questions about any website you crawl.

## 📋 Prerequisites

Before you begin, make sure you have:

- Python 3.9+ installed on your computer
- Basic knowledge of Python programming
- A text editor or IDE (like VS Code)
- A command line/terminal application

## 🔍 What You'll Build

You'll create a system that:
1. Crawls websites and extracts content
2. Stores the content in a Supabase database
3. Creates embeddings for semantic search
4. Provides a Streamlit interface to ask questions about the crawled content

## 🛠️ Step-by-Step Setup Guide

### Step 1: Clone the Repository

```bash
# Clone the repository
git clone https://github.com/ChrisVivasAI/craw4ai-tutorial-mdc-ai

# Navigate to the project directory
cd crawl4ai
```

### Step 2: Set Up a Virtual Environment

```bash
# For Windows
python -m venv crawl4ai-env
crawl4ai-env\Scripts\activate

# For macOS/Linux
python -m venv crawl4ai-env
source crawl4ai-env/bin/activate
```

### Step 3: Install Required Packages

```bash
# Install the package and its dependencies
pip install -U crawl4ai
pip install streamlit supabase-py logfire python-dotenv openai

# Run post-installation setup
crawl4ai-setup

# Verify your installation
crawl4ai-doctor
```

If you encounter any browser-related issues, install them manually:
```bash
python -m playwright install --with-deps chromium
```

### Step 4: Set Up Supabase

1. Create a free account at [Supabase](https://supabase.com/)
2. Create a new project
3. Navigate to the SQL Editor in your Supabase dashboard
4. Run the following SQL commands to set up your database:

```sql
-- Enable the pgvector extension
create extension if not exists vector;

-- Create the documentation chunks table
create table site_pages (
    id bigserial primary key,
    url varchar not null,
    chunk_number integer not null,
    title varchar not null,
    summary varchar not null,
    content text not null,
    metadata jsonb not null default '{}'::jsonb,
    embedding vector(1536),
    created_at timestamp with time zone default timezone('utc'::text, now()) not null,
    
    -- Add a unique constraint to prevent duplicate chunks for the same URL
    unique(url, chunk_number)
);

-- Create an index for better vector similarity search performance
create index on site_pages using ivfflat (embedding vector_cosine_ops);

-- Create an index on metadata for faster filtering
create index idx_site_pages_metadata on site_pages using gin (metadata);

-- Create a function to search for documentation chunks
create function match_site_pages (
  query_embedding vector(1536),
  match_count int default 10,
  filter jsonb DEFAULT '{}'::jsonb
) returns table (
  id bigint,
  url varchar,
  chunk_number integer,
  title varchar,
  summary varchar,
  content text,
  metadata jsonb,
  similarity float
)
language plpgsql
as $$
#variable_conflict use_column
begin
  return query
  select
    id,
    url,
    chunk_number,
    title,
    summary,
    content,
    metadata,
    1 - (site_pages.embedding <=> query_embedding) as similarity
  from site_pages
  where metadata @> filter
  order by site_pages.embedding <=> query_embedding
  limit match_count;
end;
$$;

-- Enable RLS on the table
alter table site_pages enable row level security;

-- Create a policy that allows anyone to read
create policy "Allow public read access"
  on site_pages
  for select
  to public
  using (true);
```

5. After running the SQL, go to Project Settings > API to find your:
   - Project URL
   - API Key (use the "service_role" key for this project)

### Step 5: Set Up OpenAI API

1. Create an account at [OpenAI](https://platform.openai.com/)
2. Navigate to API Keys and create a new secret key
3. Copy your API key for later use

### Step 6: Create Environment Variables

Create a file named `.env` in your project root with the following content:

```
# OpenAI API Key
OPENAI_API_KEY=your_openai_api_key_here

# Supabase Configuration
SUPABASE_URL=your_supabase_project_url_here
SUPABASE_SERVICE_KEY=your_supabase_service_role_key_here

# LLM Model (you can change this if needed)
LLM_MODEL=gpt-4o-mini
```

Replace the placeholder values with your actual API keys and URLs.

### Step 7: Crawl Your First Website

Now you're ready to crawl a website! Go to the crawl4ai-expert folder and click on the file named `test.py` with the following content:

```python
import asyncio
from crawl4ai import AsyncWebCrawler
from crawl4ai.async_configs import BrowserConfig, CrawlerRunConfig

async def main():
    browser_config = BrowserConfig(
        headless=False,  # Show browser for debugging
        viewport_width=1366,
        viewport_height=768,
        text_mode=False,  # Keep images enabled
        java_script_enabled=True,
        verbose=True
    )
      # Default browser configuration
    run_config = CrawlerRunConfig(
    word_count_threshold=50,        # Minimum words per content block
    exclude_external_links=True,    # Remove external links
    remove_overlay_elements=True,   # Remove popups/modals
    process_iframes=True           # Process iframe content
)   # Default crawl run configuration

    async with AsyncWebCrawler(config=browser_config) as crawler:
        result = await crawler.arun(
            url="https://www.nbcnews.com/artificial-intelligence",
            config=run_config
        )
        print(result.markdown)  # Print clean markdown content

if __name__ == "__main__":
    asyncio.run(main())
```

Run the script:
```bash
python test.py
```

This will crawl the Link next to url= you can change this if you want!

Now you have crawled your first website! Look at the terminal to see the output.

### Step 8: Crawl Documentation and Launch the Streamlit App

Before using the chat interface, you need to crawl the Crawl4AI documentation and store it in your Supabase database:

1. First, run the crawler script to populate your database:

```bash
# Navigate to the crawl4ai-expert folder
cd crawl4ai-expert

# Run the crawler script
python crawl_crawl4ai_docs.py
```

This script will:
- Crawl the Crawl4AI documentation website
- Generate embeddings for each content chunk using OpenAI
- Store the content and embeddings in your Supabase database
- Display progress in the terminal as it crawls and saves content

You'll see the crawler working in the terminal, showing each URL being processed and chunks being saved to Supabase. This process creates a semantic search index that the AI agent will use to answer your questions.

2. After the crawling is complete, launch the Streamlit app:

```bash
# Make sure you're in the crawl4ai-expert folder
streamlit run streamlit_ui.py
```

This will open a web interface where you can:
- Ask questions about Crawl4AI
- Get detailed answers based on the documentation
- Learn how to use Crawl4AI for your own web scraping projects

The AI agent uses the embeddings stored in your Supabase database to find relevant information and provide accurate answers to your questions about web scraping with Crawl4AI.

## 🧩 Understanding the Project Structure

- `crawl_crawl4ai_docs.py`: Script to crawl websites and store content in Supabase
- `crawl4ai_expert.py`: Contains the AI agent that answers questions about crawled content
- `streamlit_ui.py`: Web interface for interacting with the AI agent
- `requirements.txt`: List of Python packages needed for the project

## 🔄 Customizing the Crawler

To crawl a different website, modify the `crawl_crawl4ai_docs.py` file:

1. Change the `sitemap_url` variable in the `get_crawl4ai_docs_urls()` function
2. Update the `source` value in the `metadata` dictionary to reflect your new website

## 🚀 Next Steps

Once you're comfortable with the basic setup, you can:

1. Crawl multiple websites and build a knowledge base
2. Customize the AI agent's system prompt in `crawl4ai_expert.py`
3. Enhance the Streamlit UI with additional features
4. Experiment with different LLM models by changing the `LLM_MODEL` environment variable

## 🤔 Troubleshooting

- **Browser Issues**: If you encounter browser-related errors, try running `python -m playwright install --with-deps chromium`
- **Database Errors**: Check your Supabase credentials and make sure the SQL setup was successful
- **API Key Errors**: Verify your OpenAI API key is correct and has sufficient credits
- **Import Errors**: Make sure all required packages are installed with `pip install -r requirements.txt`

## 📚 Resources

- [Crawl4AI Documentation](https://docs.crawl4ai.com/)
- [Supabase Documentation](https://supabase.com/docs)
- [OpenAI API Documentation](https://platform.openai.com/docs/api-reference)
- [Streamlit Documentation](https://docs.streamlit.io/)

## 🙏 Acknowledgements

This project is based on [Crawl4AI](https://github.com/unclecode/crawl4ai), an open-source web crawler created by [unclecode](https://github.com/unclecode).

Happy crawling and learning! 🕸️🚀
