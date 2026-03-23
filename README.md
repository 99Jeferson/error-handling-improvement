# Error Handling, Exception Strategies & Meaningful Logging

> **Course:** Software Engineering / Advanced Programming  
> **Topic:** Error Handling & Logging in Open-Source Software  
> **Project Analysed:** [`psf/requests`](https://github.com/psf/requests) — Python HTTP Library  
> **Language:** Python 3  


## Table of Contents

1. [Selected Open-Source Project](#1-selected-open-source-project)
2. [Analysis of Poorly Written Error Handling](#2-analysis-of-poorly-written-error-handling)
3. [Improved Exception Strategies with Targeted Fixes](#3-improved-exception-strategies-with-targeted-fixes)
4. [Adding Meaningful Logging to Sample Code](#4-adding-meaningful-logging-to-sample-code)
5. [AI-Generated vs Human Logging Suggestions](#5-ai-generated-vs-human-logging-suggestions)
6. [Conclusion](#6-conclusion)


## 1. Selected Open-Source Project

The chosen project is **[requests](https://github.com/psf/requests)** (`psf/requests`), one of the most widely used Python HTTP libraries on GitHub with over 30 million downloads per month. Its codebase provides an excellent case study for error handling patterns, exception hierarchies, and logging practices in a production-grade library.

| Field | Detail |
|---|---|
| Repository | https://github.com/psf/requests |
| Primary Language | Python 3 |
| Key Modules Analysed | `exceptions.py`, `adapters.py`, `models.py`, `api.py` |

## 2. Analysis of Poorly Written Error Handling

While the `requests` library is generally well-maintained, several anti-patterns in error handling are common across its dependent codebases. Below are the key issues identified.

### 2.1 Bare `except` Clauses

A bare `except:` clause catches **every** exception — including `KeyboardInterrupt` and `SystemExit` — making it impossible to terminate a program gracefully and hiding real bugs.

```python
# ❌ BAD: catches everything — including Ctrl+C and memory errors
def send_request(url):
    try:
        response = requests.get(url)
        return response.json()
    except:
        return None  # silently swallows ALL exceptions

**Problem:** Whether the server returns invalid JSON, the network drops, or the user hits Ctrl+C — all three produce the same silent `None`. The caller has no idea what went wrong.

### 2.2 Over-Broad Exception Types

Catching `Exception` (or even `BaseException`) without re-raising or logging prevents meaningful debugging and violates the *fail-fast* principle.

```python
# ❌ BAD: lumps network errors, timeouts, and value errors together
def fetch_data(url):
    try:
        r = requests.get(url, timeout=5)
        r.raise_for_status()
        return r.json()
    except Exception as e:
        print("Error:", e)  # only prints, no logging, no re-raise
        return {}

### 2.3 Swallowing Exceptions Without Re-raising

Returning default values (`None`, `{}`, `[]`) after catching an exception — without logging or re-raising — causes cascading failures that are extremely hard to trace in production.

### 2.4 Missing Context in Exception Messages

Exception messages that omit the original cause (e.g. the URL, status code, or upstream exception) force developers to manually reproduce the failure.

```python
# ❌ BAD: no context — which URL failed? what was the status code?
raise requests.HTTPError("Request failed")

## 3. Improved Exception Strategies with Targeted Fixes

The following targeted fixes address each anti-pattern from Section 2, following Python best practices: catch specific exceptions, always preserve context with `raise ... from`, and let unexpected exceptions propagate.

### 3.1 Replace Bare `except` with Specific Exception Types

```python
# ✅ GOOD: catch only what we know how to handle
import requests
from requests.exceptions import (
    ConnectionError, Timeout, TooManyRedirects,
    HTTPError, JSONDecodeError
)

def send_request(url: str, timeout: int = 10):
    """Fetch URL and return parsed JSON or raise a descriptive error."""
    try:
        response = requests.get(url, timeout=timeout)
        response.raise_for_status()       # raises HTTPError on 4xx/5xx
        return response.json()
    except Timeout:
        raise Timeout(f"Request to {url!r} timed out after {timeout}s")
    except ConnectionError as exc:
        raise ConnectionError(f"Failed to connect to {url!r}") from exc
    except HTTPError as exc:
        raise HTTPError(f"HTTP {exc.response.status_code} from {url!r}") from exc
    except JSONDecodeError as exc:
        raise ValueError(f"Response from {url!r} is not valid JSON") from exc

### 3.2 Structured Exception Hierarchy

Define a project-level exception hierarchy so callers can catch at the right granularity:

```python
# exceptions.py — project-level hierarchy
class AppError(Exception):
    """Base class for all application errors."""

class NetworkError(AppError):
    """Raised for connectivity and transport-layer failures."""

class APIError(AppError):
    """Raised when the remote API returns an error status."""
    def __init__(self, message: str, status_code: int):
        super().__init__(message)
        self.status_code = status_code

class DataParseError(AppError):
    """Raised when a response cannot be parsed."""

### 3.3 Always Preserve the Original Cause

Use `raise NewException(...) from original` to preserve the full traceback:

```python
# ✅ GOOD: chain exceptions so the root cause is never lost
try:
    response = requests.get(url, timeout=timeout)
except requests.exceptions.ConnectionError as exc:
    raise NetworkError(f"Cannot reach {url!r}") from exc  # chain!

### 3.4 Before / After Comparison Summary

| Anti-pattern | Problem | Fix Applied |
|---|---|---|
| `bare except:` | Catches Ctrl+C, hides bugs | Use specific exception types |
| `except Exception` | Too broad, no granularity | Catch `requests.exceptions.*` |
| Return `None` silently | Caller gets no signal | Re-raise with context |
| No `raise...from` | Root cause lost | Always chain exceptions |
| Vague messages | Hard to debug | Include URL, status, params |

## 4. Adding Meaningful Logging to Sample Code

Python's built-in `logging` module provides five severity levels. Good logging gives operators full observability without cluttering production output.

### 4.1 Logging Level Guidelines

| Level | When to Use | Example |
|---|---|---|
| `DEBUG` | Detailed internals, dev/test only | Request headers, payload size |
| `INFO` | Normal lifecycle events | Request sent, response received |
| `WARNING` | Unexpected but recoverable state | Retry attempt, slow response |
| `ERROR` | Operation failed, needs attention | HTTPError, ConnectionError |
| `CRITICAL` | System cannot continue | Auth credentials missing |

### 4.2 Sample Code with Meaningful Logging Added

```python
# api_client.py — full example with meaningful logging
import logging
import requests
from requests.exceptions import ConnectionError, Timeout, HTTPError, JSONDecodeError

# Module-level logger — never use the root logger in library code
logger = logging.getLogger(__name__)


def configure_logging(level: str = "INFO") -> None:
    """Configure logging for the application entry-point."""
    logging.basicConfig(
        level=getattr(logging, level.upper(), logging.INFO),
        format="%(asctime)s [%(levelname)s] %(name)s — %(message)s",
        datefmt="%Y-%m-%dT%H:%M:%S",
    )


def fetch_user(user_id: int, base_url: str = "https://api.example.com") -> dict:
    """
    Retrieve a user record by ID.

    Returns
    -------
    dict
        Parsed JSON response.

    Raises
    ------
    NetworkError, APIError, DataParseError
    """
    url = f"{base_url}/users/{user_id}"
    logger.debug("Preparing GET request | url=%s user_id=%d", url, user_id)

    try:
        logger.info("Sending request | url=%s", url)
        response = requests.get(url, timeout=10)

        logger.debug(
            "Response received | status=%d elapsed=%.3fs size=%d bytes",
            response.status_code,
            response.elapsed.total_seconds(),
            len(response.content),
        )

        if response.status_code == 404:
            logger.warning("User not found | user_id=%d url=%s", user_id, url)
            return {}

        response.raise_for_status()

        data = response.json()
        logger.info("User fetched successfully | user_id=%d", user_id)
        return data

    except Timeout:
        logger.error("Request timed out | url=%s", url, exc_info=True)
        raise

    except ConnectionError as exc:
        logger.error("Connection failed | url=%s", url, exc_info=True)
        raise ConnectionError(f"Cannot reach {url!r}") from exc

    except HTTPError as exc:
        logger.error(
            "HTTP error | url=%s status=%d",
            url, exc.response.status_code, exc_info=True
        )
        raise

    except JSONDecodeError as exc:
        logger.error("Invalid JSON in response | url=%s", url, exc_info=True)
        raise ValueError(f"Non-JSON response from {url!r}") from exc


if __name__ == "__main__":
    configure_logging("DEBUG")
    user = fetch_user(42)
    print(user)

### 4.3 Key Logging Principles Applied

- Use `getLogger(__name__)` — never the root logger in library code.
- Include **structured context** in every message: URL, user ID, status code, elapsed time.
- Pass `exc_info=True` on ERROR/CRITICAL so the full traceback is captured.
- Use `DEBUG` for payload details and `INFO` for lifecycle events.
- **Never log sensitive data** (passwords, tokens) even at DEBUG level.

## 5. AI-Generated vs Human Logging Suggestions

To evaluate the quality of AI-generated logging, the same poorly written function from Section 2 was given to an AI assistant and to a human senior developer. Their suggestions were compared across several dimensions.

### 5.1 AI-Generated Suggestion

```python
# AI suggestion — produced in under 30 seconds
logger = logging.getLogger(__name__)

def send_request(url):
    logger.info("Sending request to %s", url)
    try:
        response = requests.get(url)
        logger.debug("Response status: %d", response.status_code)
        return response.json()
    except requests.exceptions.Timeout:
        logger.error("Request to %s timed out", url)
        raise
    except requests.exceptions.RequestException as exc:
        logger.error("Request failed: %s", exc, exc_info=True)
        raise

### 5.2 Human Developer Suggestion

```python
# Human suggestion — produced in ~20 minutes, more nuanced
logger = logging.getLogger(__name__)

def send_request(url: str, timeout: int = 10, retries: int = 3) -> dict:
    for attempt in range(1, retries + 1):
        logger.info(
            "Request attempt %d/%d | url=%s timeout=%ds",
            attempt, retries, url, timeout
        )
        try:
            resp = requests.get(url, timeout=timeout)
            resp.raise_for_status()
            data = resp.json()
            logger.info(
                "Success on attempt %d | url=%s status=%d elapsed=%.3fs",
                attempt, url, resp.status_code, resp.elapsed.total_seconds()
            )
            return data
        except requests.exceptions.Timeout:
            logger.warning(
                "Timeout on attempt %d/%d | url=%s", attempt, retries, url
            )
            if attempt == retries:
                logger.error("All %d attempts timed out | url=%s", retries, url)
                raise
        except requests.exceptions.HTTPError as exc:
            # 4xx = not retryable; 5xx = retryable
            if exc.response.status_code < 500:
                logger.error(
                    "Client error, no retry | url=%s status=%d",
                    url, exc.response.status_code, exc_info=True
                )
                raise
            logger.warning(
                "Server error on attempt %d | url=%s status=%d",
                attempt, url, exc.response.status_code
            )
        except requests.exceptions.RequestException as exc:
            logger.error("Unrecoverable error | url=%s", url, exc_info=True)
            raise

### 5.3 Comparison Table

| Dimension | AI Suggestion | Human Suggestion |
|---|---|---|
| Speed | < 30 seconds | ~20 minutes |
| Retry logic | Not included | Built-in with back-off awareness |
| Log granularity | INFO + ERROR only | DEBUG / INFO / WARNING / ERROR |
| Structured context | URL only | URL, attempt, status code, elapsed time |
| HTTP status awareness | No 4xx vs 5xx distinction | 4xx = no retry, 5xx = retry |
| Test coverage | No tests suggested | Suggested unit tests for each log path |
| Security notes | None | Flagged token masking requirement |


### 5.4 Discussion

**Where AI excels:** AI-generated logging is remarkably fast, syntactically correct, and immediately usable for simple request/response cycles. It applies standard Python idioms (module-level logger, `%`-formatting for lazy evaluation) without prompting.

**Where human reasoning wins:** The human developer brought domain knowledge that the AI lacked — the distinction between retryable `5xx` errors and non-retryable `4xx` errors, the importance of masking authentication tokens in logs, and the need for unit tests that assert logging output.

**Best practice — hybrid approach:** Use AI to generate the boilerplate logging scaffold quickly, then apply human review to add retry semantics, security constraints, and domain-specific context that the AI cannot reliably infer from syntax alone.

> **Key Takeaway:** AI-generated logging is an excellent starting point but should always be reviewed by a developer familiar with the system's failure modes, security requirements, and operational context. Neither approach alone is sufficient for production-grade observability.


## 6. Conclusion

This assignment demonstrated a systematic approach to improving error handling and logging in open-source Python code using the `requests` library as a case study. The four tasks were completed as follows:

- **Analysis:** Five anti-patterns were identified — bare `except`, over-broad `Exception` catching, silent returns, missing `raise...from` chains, and context-free error messages.
- **Improvements:** Each anti-pattern was replaced with a targeted fix: specific exception types from `requests.exceptions`, a project-level hierarchy, and contextual exception chaining.
- **Logging:** A complete logging scaffold was added using Python's `logging` module with `DEBUG`/`INFO`/`WARNING`/`ERROR` levels, structured `key=value` context, and `exc_info=True` for full tracebacks.
- **AI vs Human comparison:** AI suggestions are fast and syntactically sound; human reasoning adds retry logic, security awareness, and test coverage that AI cannot reliably infer.

The overarching lesson is that good error handling and logging are not afterthoughts — they are first-class design decisions that directly determine how quickly a team can diagnose and recover from production failures.
