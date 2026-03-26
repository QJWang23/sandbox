# arXiv Paper Tracker Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build an automated paper tracking system that fetches arXiv papers, filters by keywords/authors, analyzes with Claude, and delivers reports via Git and Feishu webhook.

**Architecture:** Four-layer system: (1) Python collectors fetch data from arXiv/Semantic Scholar/GitHub/Community, (2) Python filters score and classify items, (3) Claude Skill generates deep technical summaries, (4) Distributor saves to Git and pushes to Feishu.

**Tech Stack:** Python 3.9+, arxiv library, requests, PyYAML, pytest, Claude Code Skill

---

## Task 1: Project Skeleton Setup

**Files:**
- Create: `arxiv-tracker/`
- Create: `arxiv-tracker/requirements.txt`
- Create: `arxiv-tracker/pytest.ini`
- Create: `arxiv-tracker/.gitignore`

**Step 1: Create project directory structure**

```bash
mkdir -p arxiv-tracker/{collectors,filters,reports/{daily,weekly},data/{arxiv,github,community,filtered},config,skills,tests}
```

**Step 2: Create requirements.txt**

```text
arxiv>=2.1.0
requests>=2.31.0
PyYAML>=6.0
pytest>=7.0.0
python-dateutil>=2.8.0
```

**Step 3: Create pytest.ini**

```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_functions = test_*
```

**Step 4: Create .gitignore**

```text
__pycache__/
*.pyc
.pytest_cache/
.env
*.egg-info/
data/*.json
!data/.gitkeep
```

**Step 5: Create .gitkeep files for empty directories**

```bash
touch arxiv-tracker/data/{arxiv,github,community,filtered}/.gitkeep
touch arxiv-tracker/reports/{daily,weekly}/.gitkeep
```

**Step 6: Initialize as Python package**

```bash
touch arxiv-tracker/__init__.py
touch arxiv-tracker/collectors/__init__.py
touch arxiv-tracker/filters/__init__.py
touch arxiv-tracker/tests/__init__.py
```

**Step 7: Commit**

```bash
git add arxiv-tracker/
git commit -m "feat: initialize arxiv-tracker project skeleton"
```

---

## Task 2: Data Models

**Files:**
- Create: `arxiv-tracker/models.py`
- Create: `arxiv-tracker/tests/test_models.py`

**Step 1: Write the failing test**

```python
# arxiv-tracker/tests/test_models.py
from models import PaperItem

def test_paper_item_creation():
    item = PaperItem(
        source="arxiv",
        id="2503.12345",
        title="Test Paper",
        authors=["Author One", "Author Two"],
        date="2026-03-26",
        url="https://arxiv.org/abs/2503.12345",
        summary="This is a test abstract.",
        tags=["inference-opt"],
        heat_score=0,
        priority="normal"
    )
    assert item.source == "arxiv"
    assert item.id == "2503.12345"
    assert item.priority == "normal"

def test_paper_item_to_dict():
    item = PaperItem(
        source="arxiv",
        id="2503.12345",
        title="Test Paper",
        authors=["Author One"],
        date="2026-03-26",
        url="https://arxiv.org/abs/2503.12345",
        summary="Abstract",
        tags=[],
        heat_score=50,
        priority="high"
    )
    d = item.to_dict()
    assert d["source"] == "arxiv"
    assert d["heat_score"] == 50

def test_paper_item_from_dict():
    d = {
        "source": "github",
        "id": "release-123",
        "title": "v1.0.0 Release",
        "authors": ["Kubernetes"],
        "date": "2026-03-26",
        "url": "https://github.com/...",
        "summary": "Release notes",
        "tags": ["k8s-infra"],
        "heat_score": 30,
        "priority": "normal"
    }
    item = PaperItem.from_dict(d)
    assert item.source == "github"
    assert item.tags == ["k8s-infra"]
```

**Step 2: Run test to verify it fails**

```bash
cd arxiv-tracker && python -m pytest tests/test_models.py -v
```
Expected: FAIL with "ModuleNotFoundError: No module named 'models'"

**Step 3: Write minimal implementation**

```python
# arxiv-tracker/models.py
from dataclasses import dataclass, field, asdict
from typing import List, Optional
import json

@dataclass
class PaperItem:
    """Unified data model for all collected items."""
    source: str  # arxiv, github, nvidia_blog, hackernews
    id: str
    title: str
    authors: List[str]
    date: str
    url: str
    summary: str
    tags: List[str] = field(default_factory=list)
    heat_score: int = 0
    priority: str = "normal"  # normal, high, critical

    def to_dict(self) -> dict:
        return asdict(self)

    @classmethod
    def from_dict(cls, d: dict) -> "PaperItem":
        return cls(**d)

    def to_json(self) -> str:
        return json.dumps(self.to_dict(), ensure_ascii=False, indent=2)
```

**Step 4: Run test to verify it passes**

```bash
cd arxiv-tracker && python -m pytest tests/test_models.py -v
```
Expected: PASS (3 tests)

**Step 5: Commit**

```bash
git add arxiv-tracker/models.py arxiv-tracker/tests/test_models.py
git commit -m "feat: add PaperItem data model"
```

---

## Task 3: Configuration Files

**Files:**
- Create: `arxiv-tracker/config/sources.yaml`
- Create: `arxiv-tracker/config/watchers.yaml`
- Create: `arxiv-tracker/config/feishu.yaml`
- Create: `arxiv-tracker/config_loader.py`
- Create: `arxiv-tracker/tests/test_config_loader.py`

**Step 1: Write the failing test**

```python
# arxiv-tracker/tests/test_config_loader.py
from config_loader import load_config

def test_load_sources_config():
    config = load_config("sources")
    assert "arxiv" in config
    assert config["arxiv"]["base_url"] == "http://export.arxiv.org/api/query"

def test_load_watchers_config():
    config = load_config("watchers")
    assert "arxiv_queries" in config
    assert "cs.CL" in config["arxiv_queries"]["categories"]

def test_load_feishu_config():
    config = load_config("feishu")
    assert "webhook_url" in config
```

**Step 2: Run test to verify it fails**

```bash
cd arxiv-tracker && python -m pytest tests/test_config_loader.py -v
```
Expected: FAIL

**Step 3: Create config files**

```yaml
# arxiv-tracker/config/sources.yaml
arxiv:
  base_url: "http://export.arxiv.org/api/query"
  page_size: 100
  rate_limit: 3

semantic_scholar:
  base_url: "https://api.semanticscholar.org/graph/v1"
  api_key: "${SEMANTIC_SCHOLAR_API_KEY}"

github:
  token: "${GITHUB_TOKEN}"
  repos:
    - "kubernetes/kubernetes"
    - "kubernetes/enhancements"
    - "cncf/toc"

community:
  hackernews:
    base_url: "https://hacker-news.firebaseio.com/v0"
  reddit:
    subreddits:
      - "MachineLearning"
      - "LocalLLaMA"
```

```yaml
# arxiv-tracker/config/watchers.yaml
arxiv_queries:
  categories:
    - "cs.CL"
    - "cs.AI"
    - "cs.LG"
    - "cs.AR"
    - "cs.DC"
  keywords:
    tier1_critical:
      - "speculative decoding"
      - "flash attention"
      - "ring attention"
      - "continuous batching"
    tier2_high:
      - "inference optimization"
      - "long context"
      - "multimodal"
      - "mixture of experts"
      - "quantization"
      - "KV cache"
    tier3_normal:
      - "language model"
      - "transformer"
      - "attention mechanism"

watchers:
  authors:
    - name: "Tri Dao"
    - name: "Junxian He"
    - org: "DeepSpeed Team"
    - org: "Google DeepMind"
    - org: "Anthropic"
    - org: "vLLM Team"
    - org: "Hugging Face"

tag_keywords:
  inference-opt:
    - "quantization"
    - "speculative decoding"
    - "KV cache"
    - "continuous batching"
    - "inference optimization"
  architecture:
    - "mixture of experts"
    - "MoE"
    - "attention mechanism"
  multimodal:
    - "multimodal"
    - "vision language"
  long-context:
    - "long context"
    - "ring attention"
    - "context length"
  k8s-infra:
    - "kubernetes"
    - "container"
    - "orchestration"
  hw-accel:
    - "NVIDIA"
    - "GPU"
    - "CXL"
    - "acceleration"
```

```yaml
# arxiv-tracker/config/feishu.yaml
webhook_url: "${FEISHU_WEBHOOK_URL}"
message_templates:
  instant:
    title: "🚀 论文即时推送"
    color: "blue"
  weekly:
    title: "📊 本周论文汇总"
    color: "green"
```

**Step 4: Write config loader**

```python
# arxiv-tracker/config_loader.py
import os
import yaml
from pathlib import Path

CONFIG_DIR = Path(__file__).parent / "config"

def load_config(name: str) -> dict:
    """Load a YAML config file by name (without .yaml extension)."""
    path = CONFIG_DIR / f"{name}.yaml"
    with open(path, "r", encoding="utf-8") as f:
        content = f.read()
    # Expand environment variables
    expanded = os.path.expandvars(content)
    return yaml.safe_load(expanded)

def get_watcher_authors() -> list:
    """Get list of watched authors/organizations."""
    config = load_config("watchers")
    return config.get("watchers", {}).get("authors", [])

def get_keyword_tiers() -> dict:
    """Get keyword tier configuration."""
    config = load_config("watchers")
    return config.get("arxiv_queries", {}).get("keywords", {})

def get_tag_keywords() -> dict:
    """Get tag-to-keywords mapping."""
    config = load_config("watchers")
    return config.get("tag_keywords", {})
```

**Step 5: Run test to verify it passes**

```bash
cd arxiv-tracker && python -m pytest tests/test_config_loader.py -v
```
Expected: PASS

**Step 6: Commit**

```bash
git add arxiv-tracker/config/ arxiv-tracker/config_loader.py arxiv-tracker/tests/test_config_loader.py
git commit -m "feat: add configuration files and loader"
```

---

## Task 4: arXiv Collector

**Files:**
- Create: `arxiv-tracker/collectors/arxiv_collector.py`
- Create: `arxiv-tracker/tests/test_arxiv_collector.py`

**Step 1: Write the failing test**

```python
# arxiv-tracker/tests/test_arxiv_collector.py
import sys
sys.path.insert(0, '.')

from collectors.arxiv_collector import build_query, parse_arxiv_entry

def test_build_query():
    query = build_query(["cs.CL", "cs.AI"], ["inference optimization", "quantization"])
    assert "cat:cs.CL" in query
    assert "cat:cs.AI" in query
    assert "inference optimization" in query

def test_parse_arxiv_entry():
    # Mock entry for testing
    class MockAuthor:
        name = "Test Author"

    class MockEntry:
        id = "http://arxiv.org/abs/2503.12345v1"
        title = "Test Paper Title"
        summary = "This is a test abstract."
        authors = [MockAuthor()]
        published = "2026-03-26T10:00:00Z"
        categories = []

    entry = MockEntry()
    item = parse_arxiv_entry(entry)
    assert item.source == "arxiv"
    assert item.id == "2503.12345"
    assert item.title == "Test Paper Title"
    assert "Test Author" in item.authors
```

**Step 2: Run test to verify it fails**

```bash
cd arxiv-tracker && python -m pytest tests/test_arxiv_collector.py -v
```
Expected: FAIL

**Step 3: Write implementation**

```python
# arxiv-tracker/collectors/arxiv_collector.py
import arxiv
import json
import re
from datetime import datetime
from pathlib import Path
from typing import List, Optional
from models import PaperItem
from config_loader import load_config

DATA_DIR = Path(__file__).parent.parent / "data" / "arxiv"

def build_query(categories: List[str], keywords: List[str]) -> str:
    """Build arXiv API query string."""
    cat_query = " OR ".join([f"cat:{c}" for c in categories])
    kw_query = " OR ".join([f'all:"{kw}"' for kw in keywords])
    return f"({cat_query}) AND ({kw_query})"

def parse_arxiv_entry(entry) -> PaperItem:
    """Parse arxiv entry into PaperItem."""
    # Extract arXiv ID from URL
    arxiv_id = entry.id.split("/abs/")[-1].split("v")[0]

    # Parse date
    date = datetime.fromisoformat(entry.published.replace("Z", "+00:00")).strftime("%Y-%m-%d")

    # Extract authors
    authors = [a.name for a in entry.authors]

    # Clean title
    title = re.sub(r'\s+', ' ', entry.title).strip()

    # Clean summary
    summary = re.sub(r'\s+', ' ', entry.summary).strip()

    return PaperItem(
        source="arxiv",
        id=arxiv_id,
        title=title,
        authors=authors,
        date=date,
        url=f"https://arxiv.org/abs/{arxiv_id}",
        summary=summary,
        tags=[],
        heat_score=0,
        priority="normal"
    )

def fetch_papers(max_results: int = 100, days_back: int = 1) -> List[PaperItem]:
    """Fetch recent papers from arXiv."""
    config = load_config("sources")
    watchers = load_config("watchers")

    categories = watchers["arxiv_queries"]["categories"]
    all_keywords = []
    for tier_kws in watchers["arxiv_queries"]["keywords"].values():
        all_keywords.extend(tier_kws)

    query = build_query(categories, all_keywords)

    search = arxiv.Search(
        query=query,
        max_results=max_results,
        sort_by=arxiv.SortCriterion.SubmittedDate,
        sort_order=arxiv.SortOrder.Descending
    )

    items = []
    for result in search.results():
        item = parse_arxiv_entry(result)
        # Filter by date
        item_date = datetime.strptime(item.date, "%Y-%m-%d")
        cutoff = datetime.now() - __import__('datetime').timedelta(days=days_back)
        if item_date >= cutoff:
            items.append(item)

    return items

def save_papers(items: List[PaperItem], date_str: Optional[str] = None) -> Path:
    """Save papers to JSON file."""
    if date_str is None:
        date_str = datetime.now().strftime("%Y-%m-%d")

    DATA_DIR.mkdir(parents=True, exist_ok=True)
    path = DATA_DIR / f"{date_str}.json"

    data = [item.to_dict() for item in items]
    with open(path, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

    return path

def collect_arxiv(days_back: int = 1) -> List[PaperItem]:
    """Main collection function."""
    items = fetch_papers(days_back=days_back)
    save_papers(items)
    return items
```

**Step 4: Run test to verify it passes**

```bash
cd arxiv-tracker && python -m pytest tests/test_arxiv_collector.py -v
```
Expected: PASS

**Step 5: Commit**

```bash
git add arxiv-tracker/collectors/arxiv_collector.py arxiv-tracker/tests/test_arxiv_collector.py
git commit -m "feat: add arXiv collector"
```

---

## Task 5: Keyword Filter

**Files:**
- Create: `arxiv-tracker/filters/keyword_filter.py`
- Create: `arxiv-tracker/tests/test_keyword_filter.py`

**Step 1: Write the failing test**

```python
# arxiv-tracker/tests/test_keyword_filter.py
import sys
sys.path.insert(0, '.')

from filters.keyword_filter import KeywordFilter, filter_by_keywords

def test_keyword_filter_init():
    kf = KeywordFilter()
    assert "tier1_critical" in kf.tiers
    assert "speculative decoding" in kf.tiers["tier1_critical"]

def test_match_tier():
    kf = KeywordFilter()
    title = "Flash Attention 2: Faster Attention with Better Parallelism"
    summary = "We present an improved attention mechanism."

    tier = kf.match_tier(title, summary)
    assert tier == "tier1_critical"

def test_filter_by_keywords():
    items = [
        {"title": "Flash Attention Paper", "summary": "Attention optimization", "id": "1"},
        {"title": "Generic ML Paper", "summary": "Machine learning basics", "id": "2"},
    ]
    filtered = filter_by_keywords(items)
    assert len(filtered) == 1
    assert filtered[0]["id"] == "1"

def test_assign_priority():
    kf = KeywordFilter()
    assert kf.assign_priority("tier1_critical") == "critical"
    assert kf.assign_priority("tier2_high") == "high"
    assert kf.assign_priority("tier3_normal") == "normal"
    assert kf.assign_priority(None) == "normal"
```

**Step 2: Run test to verify it fails**

```bash
cd arxiv-tracker && python -m pytest tests/test_keyword_filter.py -v
```
Expected: FAIL

**Step 3: Write implementation**

```python
# arxiv-tracker/filters/keyword_filter.py
from typing import List, Optional, Dict
from config_loader import get_keyword_tiers

class KeywordFilter:
    """Filter papers by keyword tiers."""

    def __init__(self):
        self.tiers = get_keyword_tiers()
        self._build_pattern_cache()

    def _build_pattern_cache(self):
        """Build lowercase keyword sets for fast matching."""
        self.tier_patterns = {}
        for tier, keywords in self.tiers.items():
            self.tier_patterns[tier] = [kw.lower() for kw in keywords]

    def match_tier(self, title: str, summary: str) -> Optional[str]:
        """Find the highest priority tier matching the paper."""
        text = f"{title} {summary}".lower()

        # Check tiers in priority order
        for tier in ["tier1_critical", "tier2_high", "tier3_normal"]:
            for kw in self.tier_patterns.get(tier, []):
                if kw in text:
                    return tier
        return None

    def assign_priority(self, tier: Optional[str]) -> str:
        """Map tier to priority level."""
        mapping = {
            "tier1_critical": "critical",
            "tier2_high": "high",
            "tier3_normal": "normal",
        }
        return mapping.get(tier, "normal")

    def filter(self, items: List[dict]) -> List[dict]:
        """Filter items and assign tiers/priorities."""
        filtered = []
        for item in items:
            tier = self.match_tier(item.get("title", ""), item.get("summary", ""))
            if tier:
                item["keyword_tier"] = tier
                item["priority"] = self.assign_priority(tier)
                filtered.append(item)
        return filtered


def filter_by_keywords(items: List[dict]) -> List[dict]:
    """Convenience function for filtering."""
    kf = KeywordFilter()
    return kf.filter(items)
```

**Step 4: Run test to verify it passes**

```bash
cd arxiv-tracker && python -m pytest tests/test_keyword_filter.py -v
```
Expected: PASS

**Step 5: Commit**

```bash
git add arxiv-tracker/filters/keyword_filter.py arxiv-tracker/tests/test_keyword_filter.py
git commit -m "feat: add keyword filter with tier matching"
```

---

## Task 6: Classifier

**Files:**
- Create: `arxiv-tracker/filters/classifier.py`
- Create: `arxiv-tracker/tests/test_classifier.py`

**Step 1: Write the failing test**

```python
# arxiv-tracker/tests/test_classifier.py
import sys
sys.path.insert(0, '.')

from filters.classifier import Classifier, classify_item

def test_classifier_init():
    clf = Classifier()
    assert "inference-opt" in clf.tag_keywords
    assert "quantization" in clf.tag_keywords["inference-opt"]

def test_classify_item():
    item = {
        "title": "Efficient Quantization for LLM Inference",
        "summary": "We propose a new quantization method for inference optimization."
    }
    tags = classify_item(item)
    assert "inference-opt" in tags

def test_classify_multimodal():
    item = {
        "title": "Multimodal Understanding with Vision Language Models",
        "summary": "A new approach to multimodal reasoning."
    }
    tags = classify_item(item)
    assert "multimodal" in tags

def test_classify_multiple_tags():
    item = {
        "title": "Long Context Multimodal Inference",
        "summary": "Handling long context in multimodal models with attention optimization."
    }
    tags = classify_item(item)
    assert "multimodal" in tags
    assert "long-context" in tags
```

**Step 2: Run test to verify it fails**

```bash
cd arxiv-tracker && python -m pytest tests/test_classifier.py -v
```
Expected: FAIL

**Step 3: Write implementation**

```python
# arxiv-tracker/filters/classifier.py
from typing import List, Dict, Set
from config_loader import get_tag_keywords

class Classifier:
    """Classify papers by domain tags."""

    def __init__(self):
        self.tag_keywords = get_tag_keywords()
        self._build_pattern_cache()

    def _build_pattern_cache(self):
        """Build lowercase keyword sets."""
        self.tag_patterns = {}
        for tag, keywords in self.tag_keywords.items():
            self.tag_patterns[tag] = [kw.lower() for kw in keywords]

    def classify(self, title: str, summary: str) -> List[str]:
        """Classify a paper into domain tags."""
        text = f"{title} {summary}".lower()
        tags = []

        for tag, keywords in self.tag_patterns.items():
            for kw in keywords:
                if kw in text:
                    tags.append(tag)
                    break  # Only add each tag once

        return tags

    def classify_item(self, item: dict) -> dict:
        """Add tags to an item dict."""
        tags = self.classify(
            item.get("title", ""),
            item.get("summary", "")
        )
        item["tags"] = tags
        return item


def classify_item(item: dict) -> List[str]:
    """Convenience function for classification."""
    clf = Classifier()
    return clf.classify(item.get("title", ""), item.get("summary", ""))
```

**Step 4: Run test to verify it passes**

```bash
cd arxiv-tracker && python -m pytest tests/test_classifier.py -v
```
Expected: PASS

**Step 5: Commit**

```bash
git add arxiv-tracker/filters/classifier.py arxiv-tracker/tests/test_classifier.py
git commit -m "feat: add paper classifier"
```

---

## Task 7: Heat Scorer

**Files:**
- Create: `arxiv-tracker/filters/scorer.py`
- Create: `arxiv-tracker/tests/test_scorer.py`

**Step 1: Write the failing test**

```python
# arxiv-tracker/tests/test_scorer.py
import sys
sys.path.insert(0, '.')

from filters.scorer import HeatScorer, calculate_heat_score

def test_scorer_init():
    scorer = HeatScorer()
    assert scorer.weights["citation_rate"] == 0.3
    assert scorer.weights["community_heat"] == 0.3

def test_calculate_basic_score():
    item = {
        "keyword_tier": "tier1_critical",
        "citation_count": 10,
        "hn_score": 100
    }
    score = calculate_heat_score(item)
    assert score > 0
    assert score <= 100

def test_tier_contribution():
    scorer = HeatScorer()
    tier_score = scorer._tier_score("tier1_critical")
    assert tier_score == 100

    tier_score = scorer._tier_score("tier2_high")
    assert tier_score == 70

def test_should_trigger_instant_push():
    scorer = HeatScorer()
    # High score with high priority
    assert scorer.should_instant_push(heat_score=85, priority="high") == True
    # High score but low priority
    assert scorer.should_instant_push(heat_score=85, priority="normal") == False
    # Low score
    assert scorer.should_instant_push(heat_score=70, priority="high") == False
```

**Step 2: Run test to verify it fails**

```bash
cd arxiv-tracker && python -m pytest tests/test_scorer.py -v
```
Expected: FAIL

**Step 3: Write implementation**

```python
# arxiv-tracker/filters/scorer.py
from typing import Dict, Optional

class HeatScorer:
    """Calculate heat score for items."""

    # Weight configuration
    WEIGHTS = {
        "citation_rate": 0.3,
        "community_heat": 0.3,
        "author_influence": 0.2,
        "keyword_tier": 0.2,
    }

    # Tier score mapping
    TIER_SCORES = {
        "tier1_critical": 100,
        "tier2_high": 70,
        "tier3_normal": 40,
    }

    # Thresholds
    INSTANT_PUSH_THRESHOLD = 80

    def __init__(self, weights: Optional[Dict] = None):
        self.weights = weights or self.WEIGHTS.copy()

    def _tier_score(self, tier: Optional[str]) -> int:
        """Get score for keyword tier."""
        return self.TIER_SCORES.get(tier, 0)

    def _normalize(self, value: float, max_value: float) -> float:
        """Normalize value to 0-100 range."""
        if max_value <= 0:
            return 0
        return min(100, (value / max_value) * 100)

    def calculate(
        self,
        keyword_tier: Optional[str] = None,
        citation_count: int = 0,
        hn_score: int = 0,
        reddit_score: int = 0,
        author_influence: float = 0,
    ) -> int:
        """Calculate heat score (0-100)."""
        # Citation rate component (assume 100 citations = max)
        citation_score = self._normalize(citation_count, 100)

        # Community heat (HN + Reddit, assume 500 points = max)
        community_score = self._normalize(hn_score + reddit_score, 500)

        # Author influence (0-1 scale, normalized to 0-100)
        author_score = author_influence * 100

        # Keyword tier
        tier_score = self._tier_score(keyword_tier)

        # Weighted sum
        total = (
            self.weights["citation_rate"] * citation_score +
            self.weights["community_heat"] * community_score +
            self.weights["author_influence"] * author_score +
            self.weights["keyword_tier"] * tier_score
        )

        return int(round(total))

    def should_instant_push(self, heat_score: int, priority: str) -> bool:
        """Determine if item should trigger instant push."""
        return (
            heat_score > self.INSTANT_PUSH_THRESHOLD and
            priority in ["critical", "high"]
        )

    def score_item(self, item: dict) -> dict:
        """Add heat_score to item dict."""
        score = self.calculate(
            keyword_tier=item.get("keyword_tier"),
            citation_count=item.get("citation_count", 0),
            hn_score=item.get("hn_score", 0),
            reddit_score=item.get("reddit_score", 0),
            author_influence=item.get("author_influence", 0),
        )
        item["heat_score"] = score
        return item


def calculate_heat_score(item: dict) -> int:
    """Convenience function for scoring."""
    scorer = HeatScorer()
    return scorer.calculate(
        keyword_tier=item.get("keyword_tier"),
        citation_count=item.get("citation_count", 0),
        hn_score=item.get("hn_score", 0),
        reddit_score=item.get("reddit_score", 0),
        author_influence=item.get("author_influence", 0),
    )
```

**Step 4: Run test to verify it passes**

```bash
cd arxiv-tracker && python -m pytest tests/test_scorer.py -v
```
Expected: PASS

**Step 5: Commit**

```bash
git add arxiv-tracker/filters/scorer.py arxiv-tracker/tests/test_scorer.py
git commit -m "feat: add heat scorer with instant push logic"
```

---

## Task 8: Feishu Notifier

**Files:**
- Create: `arxiv-tracker/notifiers/feishu.py`
- Create: `arxiv-tracker/tests/test_feishu.py`

**Step 1: Write the failing test**

```python
# arxiv-tracker/tests/test_feishu.py
import sys
sys.path.insert(0, '.')

from notifiers.feishu import build_instant_card, build_weekly_card

def test_build_instant_card():
    item = {
        "title": "Test Paper",
        "summary": "This is a test abstract for the paper.",
        "url": "https://arxiv.org/abs/2503.12345",
        "heat_score": 85,
    }
    report_url = "https://github.com/user/repo/blob/main/reports/daily/2026-03-26.md"

    card = build_instant_card(item, report_url)

    assert card["msg_type"] == "interactive"
    assert "🚀" in card["card"]["header"]["title"]["content"]
    assert "Test Paper" in str(card)

def test_build_weekly_card():
    stats = {
        "total": 25,
        "categories": {
            "inference-opt": 10,
            "multimodal": 8,
            "architecture": 7,
        }
    }
    report_url = "https://github.com/user/repo/blob/main/reports/weekly/2026-W13.md"

    card = build_weekly_card(stats, report_url)

    assert card["msg_type"] == "interactive"
    assert "📊" in card["card"]["header"]["title"]["content"]
```

**Step 2: Run test to verify it fails**

```bash
cd arxiv-tracker && python -m pytest tests/test_feishu.py -v
```
Expected: FAIL

**Step 3: Write implementation**

```python
# arxiv-tracker/notifiers/feishu.py
import requests
from typing import Dict, Optional
import os

def build_instant_card(item: dict, report_url: str) -> dict:
    """Build Feishu message card for instant push."""
    title = item.get("title", "Unknown Title")
    summary = item.get("summary", "")[:200] + "..." if len(item.get("summary", "")) > 200 else item.get("summary", "")
    heat_score = item.get("heat_score", 0)

    return {
        "msg_type": "interactive",
        "card": {
            "header": {
                "title": {"tag": "plain_text", "content": "🚀 论文即时推送"},
                "template": "blue"
            },
            "elements": [
                {
                    "tag": "div",
                    "text": {
                        "tag": "lark_md",
                        "content": f"**{title}**\n\n{summary}\n\n🔥 热度: {heat_score}"
                    }
                },
                {
                    "tag": "action",
                    "actions": [
                        {
                            "tag": "button",
                            "text": {"tag": "plain_text", "content": "查看详情"},
                            "url": report_url,
                            "type": "primary"
                        }
                    ]
                }
            ]
        }
    }

def build_weekly_card(stats: dict, report_url: str) -> dict:
    """Build Feishu message card for weekly summary."""
    total = stats.get("total", 0)
    categories = stats.get("categories", {})

    cat_text = "\n".join([f"- **{k}**: {v} 篇" for k, v in categories.items()])

    return {
        "msg_type": "interactive",
        "card": {
            "header": {
                "title": {"tag": "plain_text", "content": "📊 本周论文汇总"},
                "template": "green"
            },
            "elements": [
                {
                    "tag": "div",
                    "text": {
                        "tag": "lark_md",
                        "content": f"**本周共收录 {total} 篇论文**\n\n{cat_text}"
                    }
                },
                {
                    "tag": "action",
                    "actions": [
                        {
                            "tag": "button",
                            "text": {"tag": "plain_text", "content": "查看完整报告"},
                            "url": report_url,
                            "type": "primary"
                        }
                    ]
                }
            ]
        }
    }

def send_to_feishu(card: dict, webhook_url: Optional[str] = None) -> bool:
    """Send message card to Feishu webhook."""
    webhook_url = webhook_url or os.environ.get("FEISHU_WEBHOOK_URL")
    if not webhook_url:
        print("Warning: No Feishu webhook URL configured")
        return False

    try:
        response = requests.post(
            webhook_url,
            json=card,
            headers={"Content-Type": "application/json"},
            timeout=10
        )
        response.raise_for_status()
        return True
    except Exception as e:
        print(f"Failed to send to Feishu: {e}")
        return False
```

**Step 4: Create notifiers directory**

```bash
mkdir -p arxiv-tracker/notifiers
touch arxiv-tracker/notifiers/__init__.py
```

**Step 5: Run test to verify it passes**

```bash
cd arxiv-tracker && python -m pytest tests/test_feishu.py -v
```
Expected: PASS

**Step 6: Commit**

```bash
git add arxiv-tracker/notifiers/ arxiv-tracker/tests/test_feishu.py
git commit -m "feat: add Feishu notifier with message cards"
```

---

## Task 9: Claude Skill Definition

**Files:**
- Create: `arxiv-tracker/skills/paper-analyzer.md`

**Step 1: Create the skill file**

```markdown
---
name: paper-analyzer
description: Analyze filtered papers and generate technical summaries
invocable: true
---

# Paper Analyzer Skill

Analyzes papers from the arxiv-tracker and generates detailed technical summaries.

## Usage

```
/paper-analyzer --mode=instant --date=YYYY-MM-DD
/paper-analyzer --mode=weekly --week=YYYY-WNN
```

## Process

1. **Load filtered data** from `data/filtered/{date}.json`
2. **For each paper**, generate:
   - Core technical points (preserving key terminology)
   - Innovation points (2-3 sentences on what's novel)
   - Impact on your focus areas (inference optimization, K8s infra, hardware acceleration)
   - Actionable follow-up points
3. **Output** structured markdown report

## Report Template

```markdown
## {{title}}

**Source**: {{source}} | **Date**: {{date}} | **Heat Score**: {{heat_score}}

### Core Technical Points
- [Specific technical point 1, preserving original terminology]
- [Specific technical point 2, preserving original terminology]

### Innovation
[2-3 sentences explaining the essential difference from existing methods]

### Impact on Focus Areas
- **Inference Optimization**: [Specific impact]
- **K8s Infrastructure**: [Specific impact] (if applicable)
- **Hardware Acceleration**: [Specific impact] (if applicable)

### Follow-up Points
- [ ] [Actionable technical point or experiment direction]
- [ ] [Related papers to read further]

---
arXiv: {{arxiv_id}} | [Link]({{url}})
```

## Key Requirements

- **DO NOT over-summarize** - preserve technical details and terminology
- **Focus on what's new** - highlight the innovation, not background
- **Connect to user's domain** - explicitly analyze impact on inference optimization, K8s, and hardware acceleration
- **Be actionable** - suggest concrete follow-up actions

## Output Location

- Instant reports: `reports/daily/{date}-instant.md`
- Weekly reports: `reports/weekly/{year}-W{week}.md`
```

**Step 2: Commit**

```bash
git add arxiv-tracker/skills/paper-analyzer.md
git commit -m "feat: add paper-analyzer Claude skill definition"
```

---

## Task 10: Main Entry Point

**Files:**
- Create: `arxiv-tracker/run.py`
- Create: `arxiv-tracker/tests/test_run.py`

**Step 1: Write the failing test**

```python
# arxiv-tracker/tests/test_run.py
import sys
sys.path.insert(0, '.')

from run import load_filtered_data, filter_hot_items

def test_load_filtered_data(tmp_path):
    import json
    # Create test data
    data_file = tmp_path / "2026-03-26.json"
    data = [{"id": "1", "title": "Test", "heat_score": 85, "priority": "high"}]
    data_file.write_text(json.dumps(data))

    items = load_filtered_data(data_file)
    assert len(items) == 1
    assert items[0]["id"] == "1"

def test_filter_hot_items():
    items = [
        {"id": "1", "heat_score": 85, "priority": "high"},
        {"id": "2", "heat_score": 70, "priority": "high"},
        {"id": "3", "heat_score": 90, "priority": "normal"},
        {"id": "4", "heat_score": 95, "priority": "critical"},
    ]
    hot = filter_hot_items(items)
    assert len(hot) == 2
    assert hot[0]["id"] == "4"  # critical with high score first
    assert hot[1]["id"] == "1"  # high with score > 80
```

**Step 2: Run test to verify it fails**

```bash
cd arxiv-tracker && python -m pytest tests/test_run.py -v
```
Expected: FAIL

**Step 3: Write implementation**

```python
# arxiv-tracker/run.py
#!/usr/bin/env python3
"""
arXiv Paper Tracker - Main Entry Point

Usage:
    python run.py --mode=daily      # Daily collection and instant push
    python run.py --mode=weekly     # Weekly summary generation
"""

import argparse
import json
import sys
from datetime import datetime
from pathlib import Path
from typing import List, Dict

# Add parent to path for imports
sys.path.insert(0, str(Path(__file__).parent))

from collectors.arxiv_collector import collect_arxiv
from filters.keyword_filter import filter_by_keywords
from filters.classifier import Classifier
from filters.scorer import HeatScorer
from models import PaperItem

BASE_DIR = Path(__file__).parent
DATA_DIR = BASE_DIR / "data"
FILTERED_DIR = DATA_DIR / "filtered"
REPORTS_DIR = BASE_DIR / "reports"


def collect_all_sources(days_back: int = 1) -> List[PaperItem]:
    """Collect from all configured sources."""
    print("📚 Collecting from arXiv...")
    items = collect_arxiv(days_back=days_back)
    print(f"   Found {len(items)} papers")
    return items


def filter_and_score(items: List[PaperItem]) -> List[Dict]:
    """Apply all filters and scoring."""
    print("🔍 Filtering and scoring...")

    # Convert to dicts
    item_dicts = [i.to_dict() for i in items]

    # Keyword filter
    filtered = filter_by_keywords(item_dicts)
    print(f"   After keyword filter: {len(filtered)}")

    # Classify
    clf = Classifier()
    for item in filtered:
        clf.classify_item(item)

    # Score
    scorer = HeatScorer()
    for item in filtered:
        scorer.score_item(item)

    # Sort by heat score
    filtered.sort(key=lambda x: x["heat_score"], reverse=True)

    return filtered


def save_filtered(items: List[Dict], date_str: str = None) -> Path:
    """Save filtered items to JSON."""
    if date_str is None:
        date_str = datetime.now().strftime("%Y-%m-%d")

    FILTERED_DIR.mkdir(parents=True, exist_ok=True)
    path = FILTERED_DIR / f"{date_str}.json"

    with open(path, "w", encoding="utf-8") as f:
        json.dump(items, f, ensure_ascii=False, indent=2)

    print(f"💾 Saved filtered data to {path}")
    return path


def load_filtered_data(path: Path) -> List[Dict]:
    """Load filtered data from JSON."""
    with open(path, "r", encoding="utf-8") as f:
        return json.load(f)


def filter_hot_items(items: List[Dict]) -> List[Dict]:
    """Get items that should trigger instant push."""
    scorer = HeatScorer()
    return [
        item for item in items
        if scorer.should_instant_push(item["heat_score"], item["priority"])
    ]


def generate_report(items: List[Dict], mode: str, date_str: str = None) -> Path:
    """Generate markdown report (stub - actual analysis by Claude skill)."""
    if date_str is None:
        date_str = datetime.now().strftime("%Y-%m-%d")

    if mode == "instant":
        report_dir = REPORTS_DIR / "daily"
        filename = f"{date_str}-instant.md"
    else:
        week_num = datetime.now().isocalendar()[1]
        filename = f"{datetime.now().year}-W{week_num:02d}.md"
        report_dir = REPORTS_DIR / "weekly"

    report_dir.mkdir(parents=True, exist_ok=True)
    path = report_dir / filename

    # Generate basic report structure
    lines = [f"# {'即时推送' if mode == 'instant' else '周汇总'} - {date_str}\n"]

    for item in items:
        lines.append(f"## {item['title']}\n")
        lines.append(f"- **Source**: {item['source']}")
        lines.append(f"- **Heat Score**: {item['heat_score']}")
        lines.append(f"- **Priority**: {item['priority']}")
        lines.append(f"- **Tags**: {', '.join(item['tags'])}")
        lines.append(f"- **URL**: {item['url']}\n")
        lines.append(f"**Summary**: {item['summary'][:500]}...\n")
        lines.append("---\n")

    path.write_text("\n".join(lines), encoding="utf-8")
    print(f"📝 Generated report: {path}")
    return path


def main(mode: str, days_back: int = 1):
    """Main entry point."""
    print(f"🚀 arXiv Paper Tracker - {mode} mode\n")

    date_str = datetime.now().strftime("%Y-%m-%d")

    # Step 1: Collect
    items = collect_all_sources(days_back=days_back)

    if not items:
        print("No items collected. Exiting.")
        return

    # Step 2: Filter and score
    filtered = filter_and_score(items)

    if not filtered:
        print("No items passed filtering. Exiting.")
        return

    # Step 3: Save filtered data
    save_filtered(filtered, date_str)

    # Step 4: Process based on mode
    if mode == "daily":
        hot_items = filter_hot_items(filtered)
        if hot_items:
            print(f"🔥 {len(hot_items)} items qualify for instant push")
            generate_report(hot_items, "instant", date_str)
            print("\n⚠️  Run Claude skill for deep analysis:")
            print(f"   /paper-analyzer --mode=instant --date={date_str}")
        else:
            print("No items qualify for instant push today.")
    else:
        generate_report(filtered, "weekly")
        print("\n⚠️  Run Claude skill for deep analysis:")
        week_num = datetime.now().isocalendar()[1]
        print(f"   /paper-analyzer --mode=weekly --week={datetime.now().year}-W{week_num:02d}")

    print("\n✅ Done!")


if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="arXiv Paper Tracker")
    parser.add_argument(
        "--mode",
        choices=["daily", "weekly"],
        default="daily",
        help="Run mode: daily (instant push) or weekly (summary)"
    )
    parser.add_argument(
        "--days-back",
        type=int,
        default=1,
        help="Number of days to look back for papers"
    )
    args = parser.parse_args()
    main(args.mode, args.days_back)
```

**Step 4: Run test to verify it passes**

```bash
cd arxiv-tracker && python -m pytest tests/test_run.py -v
```
Expected: PASS

**Step 5: Commit**

```bash
git add arxiv-tracker/run.py arxiv-tracker/tests/test_run.py
git commit -m "feat: add main entry point with daily/weekly modes"
```

---

## Task 11: Integration Test

**Files:**
- Create: `arxiv-tracker/tests/test_integration.py`

**Step 1: Write integration test**

```python
# arxiv-tracker/tests/test_integration.py
"""
Integration test for the full pipeline.
Uses mock data to test the complete flow.
"""
import sys
import json
import tempfile
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent.parent))

from models import PaperItem
from filters.keyword_filter import filter_by_keywords
from filters.classifier import classify_item
from filters.scorer import HeatScorer


def test_full_pipeline():
    """Test the complete filtering pipeline."""
    # Create mock items
    items = [
        PaperItem(
            source="arxiv",
            id="2503.00001",
            title="Flash Attention 3: Fast Attention with Sparse Patterns",
            authors=["Tri Dao", "Author Two"],
            date="2026-03-26",
            url="https://arxiv.org/abs/2503.00001",
            summary="We present Flash Attention 3, a new approach to attention that achieves 2x speedup through sparse attention patterns and improved GPU utilization.",
            tags=[],
        ).to_dict(),
        PaperItem(
            source="arxiv",
            id="2503.00002",
            title="Efficient Quantization for Large Language Models",
            authors=["Some Author"],
            date="2026-03-26",
            url="https://arxiv.org/abs/2503.00002",
            summary="We propose a new quantization method that reduces memory usage by 4x while maintaining accuracy for inference optimization.",
            tags=[],
        ).to_dict(),
        PaperItem(
            source="arxiv",
            id="2503.00003",
            title="Generic Machine Learning Paper",
            authors=["Unknown Author"],
            date="2026-03-26",
            url="https://arxiv.org/abs/2503.00003",
            summary="A general overview of machine learning concepts.",
            tags=[],
        ).to_dict(),
    ]

    # Step 1: Keyword filter
    filtered = filter_by_keywords(items)
    assert len(filtered) == 2  # Only first two match keywords

    # Step 2: Classify
    for item in filtered:
        tags = classify_item(item)
        item["tags"] = tags

    # Verify classification
    assert "inference-opt" in filtered[1]["tags"]  # Quantization paper

    # Step 3: Score
    scorer = HeatScorer()
    for item in filtered:
        scorer.score_item(item)

    # First paper should have higher score (tier1 critical keyword)
    assert filtered[0]["heat_score"] > filtered[1]["heat_score"]
    assert filtered[0]["priority"] == "critical"  # Flash attention

    # Step 4: Check instant push logic
    hot_items = [
        item for item in filtered
        if scorer.should_instant_push(item["heat_score"], item["priority"])
    ]
    assert len(hot_items) >= 1  # At least the Flash Attention paper


def test_save_and_load_filtered_data():
    """Test saving and loading filtered data."""
    items = [
        PaperItem(
            source="arxiv",
            id="2503.00001",
            title="Test Paper",
            authors=["Author"],
            date="2026-03-26",
            url="https://arxiv.org/abs/2503.00001",
            summary="Abstract",
            tags=["inference-opt"],
            heat_score=85,
            priority="high"
        ).to_dict()
    ]

    with tempfile.TemporaryDirectory() as tmpdir:
        path = Path(tmpdir) / "test.json"
        path.write_text(json.dumps(items), encoding="utf-8")

        loaded = json.loads(path.read_text(encoding="utf-8"))
        assert len(loaded) == 1
        assert loaded[0]["id"] == "2503.00001"
```

**Step 2: Run test to verify it passes**

```bash
cd arxiv-tracker && python -m pytest tests/test_integration.py -v
```
Expected: PASS

**Step 3: Commit**

```bash
git add arxiv-tracker/tests/test_integration.py
git commit -m "test: add integration test for full pipeline"
```

---

## Task 12: Final Verification and Documentation

**Files:**
- Create: `arxiv-tracker/README.md`

**Step 1: Create README**

```markdown
# arXiv Paper Tracker

Automated paper tracking system for AI/LLM research with smart filtering and instant push notifications.

## Features

- **Multi-source collection**: arXiv, Semantic Scholar, GitHub, Hacker News
- **Smart filtering**: Keyword tiers, author watching, heat scoring
- **Deep analysis**: Claude-powered technical summaries
- **Instant push**: Critical papers sent immediately via Feishu

## Quick Start

```bash
# Install dependencies
pip install -r requirements.txt

# Run daily collection
python run.py --mode=daily

# Run weekly summary
python run.py --mode=weekly
```

## Configuration

1. Copy `config/sources.yaml.example` to `config/sources.yaml`
2. Set environment variables:
   - `SEMANTIC_SCHOLAR_API_KEY` (optional)
   - `GITHUB_TOKEN` (optional, for higher rate limits)
   - `FEISHU_WEBHOOK_URL` (for notifications)

3. Customize `config/watchers.yaml` with your interests:
   - Keywords in `arxiv_queries.keywords`
   - Authors/organizations in `watchers.authors`

## Project Structure

```
arxiv-tracker/
├── collectors/      # Data collection modules
├── filters/         # Filtering and scoring
├── notifiers/       # Push notifications
├── skills/          # Claude skill definitions
├── reports/         # Generated reports
├── data/            # Cached data
└── config/          # Configuration files
```

## Usage

### Daily Mode
Collects papers, filters by keywords, and identifies papers for instant push.

```bash
python run.py --mode=daily
```

### Weekly Mode
Generates a summary of all filtered papers from the past week.

```bash
python run.py --mode=weekly
```

### Claude Skill Analysis
After running the collection, use the Claude skill for deep analysis:

```
/paper-analyzer --mode=instant --date=2026-03-26
```
```

**Step 2: Run all tests**

```bash
cd arxiv-tracker && python -m pytest -v
```

**Step 3: Commit**

```bash
git add arxiv-tracker/README.md
git commit -m "docs: add README with quick start guide"
```

---

## Summary

| Task | Description | Key Files |
|------|-------------|-----------|
| 1 | Project skeleton | `arxiv-tracker/` structure |
| 2 | Data models | `models.py` |
| 3 | Configuration | `config/*.yaml`, `config_loader.py` |
| 4 | arXiv collector | `collectors/arxiv_collector.py` |
| 5 | Keyword filter | `filters/keyword_filter.py` |
| 6 | Classifier | `filters/classifier.py` |
| 7 | Heat scorer | `filters/scorer.py` |
| 8 | Feishu notifier | `notifiers/feishu.py` |
| 9 | Claude skill | `skills/paper-analyzer.md` |
| 10 | Main entry | `run.py` |
| 11 | Integration test | `tests/test_integration.py` |
| 12 | Documentation | `README.md` |