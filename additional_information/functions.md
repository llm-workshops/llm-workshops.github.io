---
parent: "Additional Information"
title: "PPI filter function"
nav_order: 3
---
## PPI filter function
In the [section on Open WebUI functions](../functions/quickstart_function.md) you saw a function for filtering PPI. Here, we give some background on PPI filtering and explain the code in detail.

### About PPI filtering
Private Personal Information (PPI) or Personal Data filtering is a critical component in any system that processes real-world user or employee information, especially when large language models or automated tools are involved. Without proper filtering, sensitive attributes such as names, email addresses, phone numbers, birth dates, or job roles can be inadvertently exposed, logged, or even leaked to downstream systems. A well-known example is the [2017 Equifax data breach](https://en.wikipedia.org/wiki/2017_Equifax_data_breach), where insufficient safeguards around access and data handling led to the exposure of highly sensitive personal information for over 140 million individuals, including Social Security numbers and birth dates. While that incident was not caused by an LLM, it illustrates how devastating improper handling of personal data can be, both legally and reputationally, and why modern AI toolchains must treat PPI as a first-class security concern.

In the context of MCP tool servers and LLM integrations, PPI filtering ensures that models only see or return information that is strictly necessary for the task. For instance, if a user asks for statistical counts of employees by role or gender, the system should never return raw emails or phone numbers, even if those fields exist in the underlying dataset.

### Code explanation
We will now go through the PPI filtering function, as seen in the [functions section](../functions/quickstart_function.md), step by step, to explain how it is structured and what each component does.

The first step is to import required packages, and define the patterns for sensitive data with regex. For an overview of how regex works, we refer to [this cheatsheet](https://www.rexegg.com/regex-quickstart.php). Here, we specify a pattern for an api key, a private key and an aws secret. We note that these patterns clearly do not cover all cases, and are mostly for illustrative purposes in the exercise. 
```python
import re
from pydantic import BaseModel
from typing import Tuple, Optional

# Simple patterns for common sensitive data
SENSITIVE_PATTERNS = {
    "api_key": re.compile(r"(api[_-]?key\s*[:=]\s*[A-Za-z0-9_\-]{16,})", re.IGNORECASE),
    "private_key": re.compile(
        r"-----BEGIN (RSA|EC|DSA)? ?PRIVATE KEY-----", re.IGNORECASE
    ),
    "aws_secret": re.compile(
        r"(aws[_-]?secret[_-]?access[_-]?key\s*[:=]\s*[A-Za-z0-9/+=]{40})",
        re.IGNORECASE,
    ),
}
```

Next, we initiate a boolean variable `CONTAINS_PPI` to `False`. We will toggle this to be true if any of our Regex patterns is detected.
```python
CONTAINS_PPI = False
```

Now, we can start with the main `Filter` class. We do not have any user-configurable options in the base case, hence the `Valves` class is empty. We initiate `self.toggle` to be true, which allows us to toggle the filter function on and off from the chat interface. We also provide a path to an icon, which will be the icon representing the filter function, in this case a shield.
```python
class Filter:
    class Valves(BaseModel):
        pass

    def __init__(self):
        # Initialize valves (optional configuration for the Filter)
        self.valves = self.Valves()
        self.toggle = True
        self.icon = """data:image/svg+xml,%3C%3Fxml%20version%3D%221.0%22%20encoding%3D%22iso-8859-1%22%3F%3E%0A%3C!--%20Uploaded%20to%3A%20SVG%20Repo%2C%20www.svgrepo.com%2C%20Generator%3A%20SVG%20Repo%20Mixer%20Tools%20--%3E%0A%3C!DOCTYPE%20svg%20PUBLIC%20%22-%2F%2FW3C%2F%2FDTD%20SVG%201.1%2F%2FEN%22%20%22http%3A%2F%2Fwww.w3.org%2FGraphics%2FSVG%2F1.1%2FDTD%2Fsvg11.dtd%22%3E%0A%3Csvg%20fill%3D%22%23000000%22%20height%3D%22800px%22%20width%3D%22800px%22%20version%3D%221.1%22%20id%3D%22Capa_1%22%20xmlns%3D%22http%3A%2F%2Fwww.w3.org%2F2000%2Fsvg%22%20xmlns%3Axlink%3D%22http%3A%2F%2Fwww.w3.org%2F1999%2Fxlink%22%20%0A%09%20viewBox%3D%220%200%20347.971%20347.971%22%20xml%3Aspace%3D%22preserve%22%3E%0A%3Cpath%20d%3D%22M317.309%2C54.367C257.933%2C54.367%2C212.445%2C37.403%2C173.98%2C0C135.519%2C37.403%2C90.033%2C54.367%2C30.662%2C54.367%0A%09c0%2C97.405-20.155%2C236.937%2C143.317%2C293.604C337.463%2C291.305%2C317.309%2C151.773%2C317.309%2C54.367z%20M162.107%2C225.773l-47.749-47.756%0A%09l21.379-21.378l26.37%2C26.376l50.121-50.122l21.378%2C21.378L162.107%2C225.773z%22%2F%3E%0A%3C%2Fsvg%3E"""
```

Next, we define the function that will scrub the PPI from a given text. We will search the text for each of the items in our `SENSITIVE_PATTERNS` dictionary, and if any sensitive text is detected we will replace it by the string `[REDACTED]`. Additionally, if we want to warn the user, we set the global parameter `CONTAINS_PPI` to true. This will be verified in the `outlet` function later, which will trigger a user warning.
```python
    def scrub_pii(self, text: str) -> str:
        """Redact sensitive information from the text."""
        global CONTAINS_PPI
        detected = False
        for _, pattern in SENSITIVE_PATTERNS.items():
            if pattern.search(text):
                detected = True
            text = pattern.sub(
                lambda m: m.group(0).split(":")[0] + ": [REDACTED]",
                text,
            )
        if detected:
            CONTAINS_PPI = True
        return text
```

Now, we define the `inlet` function, which deals with the input prompt from the user. In our exercise, we filter the user prompt for sensitive information. Hence, we apply our previously defined `scrub_ppi` function on the latest message from the user, and update the text.
```
    async def inlet(
        self, body: dict, __event_emitter__, __user__: Optional[dict] = None
    ) -> dict:
        """
        Returns:
            - possibly modified user input
            - boolean indicating whether sensitive data was detected
        """
        await __event_emitter__(
            {
                "type": "status",
                "data": {
                    "description": "Filtering PPI",
                    "done": True,
                    "hidden": False,
                },
            }
        )
        last_message = body["messages"][-1]["content"]
        filtered_input = self.scrub_pii(last_message)
        body["messages"][-1]["content"] = filtered_input
        return body
```

Last, we need to define the outlet function, which deals with the final text that will be returned to the user. In this case, we want to add a warning to the user if the `CONTAINS_PPI` variable is `True`. Note that if we would want to filter the output of the LLM for PPI, we could do that in the `outlet` function too. 
```python
    def outlet(self, body: dict, __user__: Optional[dict] = None) -> dict:
        for message in body["messages"]:
            if message["role"] == "assistant" and CONTAINS_PPI:
                message[
                    "content"
                ] += "\n\n **Note: Sensitive information has been redacted from the user's input.**"
        return body
```

_Author: [Alexander Sternfeld](https://ch.linkedin.com/in/alexander-sternfeld-93a01799)_
