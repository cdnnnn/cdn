//test_client.py
"""Smoke test: launches the server over stdio and calls every tool."""

import asyncio
import sys

from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

CALLS = [
    ("count_words", {"text": "Hello world. This is MCP! Does it work?"}),
    ("compare_text", {"text_a": '{\n  "port": 8080,\n  "debug": true\n}',
                      "text_b": '{\n  "port": 9090,\n  "debug": true\n}'}),
    ("format_json", {"text": '{"b":1,"a":[1,2]}', "sort_keys": True}),
    ("format_json", {"text": '{"a": 1,}'}),
    ("convert_case", {"text": "don't SHOUT at the o'neil family", "case": "title"}),
    ("extract_emails", {"text": "Mail a@x.com, B@y.co.in and A@X.com. Not this: foo@bar"}),
]


async def main():
    params = StdioServerParameters(command=sys.executable, args=["text_utils_server.py"])
    async with stdio_client(params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            tools = await session.list_tools()
            print("tools:", [t.name for t in tools.tools])
            for name, args in CALLS:
                result = await session.call_tool(name, args)
                print(f"\n{name}({args})\n  error={result.is_error}\n  ->", result.content[0].text)


asyncio.run(main())






//text_utils_server.py
"""Smart File & Text Utility MCP server.

Five small tools: count_words, compare_text, format_json, convert_case,
extract_emails.

Run:  python text_utils_server.py          (stdio, for local clients)
      python text_utils_server.py --http   (Streamable HTTP on :8000/mcp,
                                            for remote/custom connectors)
Deps: pip install "mcp>=2"
"""

import difflib
import json
import os
import re
import sys
from typing import Literal

from mcp.server.mcpserver import MCPServer

mcp = MCPServer("text-utils")

EMAIL_RE = re.compile(r"[A-Za-z0-9._%+-]+@[A-Za-z0-9-]+(?:\.[A-Za-z0-9-]+)*\.[A-Za-z]{2,}")
SENTENCE_END_RE = re.compile(r"[.!?]+(?=\s|$)")


@mcp.tool()
def count_words(text: str) -> dict:
    """Count words, characters and sentences in a piece of text."""
    stripped = text.strip()
    sentences = len(SENTENCE_END_RE.findall(stripped))
    if stripped and sentences == 0:
        sentences = 1  # text with no closing punctuation is still one sentence
    return {
        "words": len(text.split()),
        "characters": len(text),
        "characters_no_spaces": len(re.sub(r"\s", "", text)),
        "sentences": sentences,
        "lines": len(text.splitlines()),
    }


@mcp.tool()
def compare_text(text_a: str, text_b: str, ignore_whitespace: bool = False) -> dict:
    """Compare two texts line by line and report what changed.

    Returns a unified diff plus the added and removed lines. For JSON,
    run both inputs through format_json (sort_keys=True) first so that
    key order and indentation don't show up as differences.
    """
    a_lines = text_a.splitlines()
    b_lines = text_b.splitlines()
    if ignore_whitespace:
        a_lines = [" ".join(line.split()) for line in a_lines]
        b_lines = [" ".join(line.split()) for line in b_lines]

    diff = list(difflib.unified_diff(a_lines, b_lines, "text_a", "text_b", lineterm=""))
    added = [l[1:] for l in diff if l.startswith("+") and not l.startswith("+++")]
    removed = [l[1:] for l in diff if l.startswith("-") and not l.startswith("---")]
    return {
        "identical": not diff,
        "similarity": round(difflib.SequenceMatcher(None, a_lines, b_lines).ratio(), 3),
        "added_lines": added,
        "removed_lines": removed,
        "diff": "\n".join(diff),
    }


@mcp.tool()
def format_json(text: str, indent: int = 2, sort_keys: bool = False) -> dict:
    """Validate JSON and return it pretty-printed.

    If the input is invalid, returns valid=False with the error message
    and its line/column instead of raising.
    """
    try:
        parsed = json.loads(text)
    except json.JSONDecodeError as e:
        return {"valid": False, "error": e.msg, "line": e.lineno, "column": e.colno}
    return {
        "valid": True,
        "formatted": json.dumps(parsed, indent=indent, sort_keys=sort_keys, ensure_ascii=False),
    }


@mcp.tool()
def convert_case(text: str, case: Literal["upper", "lower", "title"]) -> str:
    """Convert text to uppercase, lowercase or title case."""
    if case == "upper":
        return text.upper()
    if case == "lower":
        return text.lower()
    # str.title() mangles apostrophes ("don't" -> "Don'T"), so capitalise per word.
    return re.sub(r"[^\W\d_]+(?:'[^\W\d_]+)*", lambda m: m.group().capitalize(), text)


@mcp.tool()
def extract_emails(text: str, unique: bool = True) -> dict:
    """Extract email addresses from text, in order of first appearance.

    With unique=True (default), duplicates are removed case-insensitively.
    """
    found = EMAIL_RE.findall(text)
    if unique:
        seen: set[str] = set()
        deduped = []
        for email in found:
            if email.lower() not in seen:
                seen.add(email.lower())
                deduped.append(email)
        found = deduped
    return {"count": len(found), "emails": found}


if __name__ == "__main__":
    if "--http" in sys.argv:
        # 0.0.0.0 so a tunnel or host platform can reach it; PORT is set by most hosts.
        mcp.run(
            "streamable-http",
            host="0.0.0.0",
            port=int(os.environ.get("PORT", 8000)),
            stateless_http=True,
            json_response=True,
        )
    else:
        mcp.run()














npx @modelcontextprotocol/inspector python text_utils_server.py
python text_utils_server.py --http
