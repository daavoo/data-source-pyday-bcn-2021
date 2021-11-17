# dataset-comida-hobby-libro

---
## Setup Repo

```
pip install -r requirements.txt
```

```
# https://dvc.org/doc/command-reference/init
dvc init
```

```
# https://dvc.org/doc/user-guide/setup-google-drive-remote
dvc remote add myremote gdrive://1BhytlB8XyBTcTNeavk9_3J1a7n_kCFdk
``` 
---
## Setup Pipeline

<details>
<summary>Create `params.yaml`</summary>

```yaml
state: open
since: 2021/1/1
until: 2021/11/17

data_folder: data
```

</details>

<details>
<summary>Create `src/get_data.py`</summary>

```py
import os
from datetime import datetime
from pathlib import Path

import fire
import yaml
from github import Github
from loguru import logger


def get_data(output_folder):
    with open("params.yaml") as f:
        params = yaml.safe_load(f)

    output_folder = Path(output_folder)
    for label in params["labels"]:
        (output_folder / label).mkdir(parents=True, exist_ok=True)

    since = datetime(*map(int, params["since"].split("/")))
    until = datetime(*map(int, params["until"].split("/")))
    logger.info(f"Getting issue labels since {since} until {until}")

    logger.info("Initializing Github")
    if os.environ.get("GITHUB_TOKEN"):
        g = Github(os.environ["GITHUB_TOKEN"])
    else:
        g = Github()

    logger.info(f"Querying repo: {params['repo']}")
    repo = g.get_repo(params["repo"])

    for issue in repo.get_issues(state=params["state"], since=since):
        issue_labels = [
            x.name for x in issue.labels if x.name in params["labels"]
        ]
    
        if (
            issue.pull_request 
            or issue.created_at > until
            or len(issue_labels) != 1
        ):
            logger.debug(f"Skipping issue: {issue.title}")
            logger.debug(f"Created at: {issue.created_at}")
            logger.debug(f"Labels: {issue.labels}")
            continue
    
        label = str(issue_labels[0])
        logger.info(f"TITLE:\n{issue.title}")
        logger.info(f"BODY:\n{issue.body}")
        logger.info(f"LABEL:\n{label}")

        output_file = output_folder / label / f"{issue.number}.txt"
        output_file.write_text(f"{issue.title}\n{issue.body}")


if __name__ == "__main__":
    fire.Fire(get_data)
```

</details>


<details>
<summary>Create `src/compute_metrics.py`</summary>

```py
import os
from datetime import datetime
from pathlib import Path

import fire
import yaml
from github import Github
from loguru import logger


def get_data(output_folder):
    with open("params.yaml") as f:
        params = yaml.safe_load(f)

    output_folder = Path(output_folder)
    for label in params["labels"]:
        (output_folder / label).mkdir(parents=True, exist_ok=True)

    since = datetime(*map(int, params["since"].split("/")))
    until = datetime(*map(int, params["until"].split("/")))
    logger.info(f"Getting issue labels since {since} until {until}")

    logger.info("Initializing Github")
    if os.environ.get("GITHUB_TOKEN"):
        g = Github(os.environ["GITHUB_TOKEN"])
    else:
        g = Github()

    logger.info(f"Querying repo: {params['repo']}")
    repo = g.get_repo(params["repo"])

    for issue in repo.get_issues(state=params["state"], since=since):
        issue_labels = [
            x.name for x in issue.labels if x.name in params["labels"]
        ]
    
        if (
            issue.pull_request 
            or issue.created_at > until
            or len(issue_labels) != 1
        ):
            logger.debug(f"Skipping issue: {issue.title}")
            logger.debug(f"Created at: {issue.created_at}")
            logger.debug(f"Labels: {issue.labels}")
            continue
    
        label = str(issue_labels[0])
        logger.info(f"TITLE:\n{issue.title}")
        logger.info(f"BODY:\n{issue.body}")
        logger.info(f"LABEL:\n{label}")

        output_file = output_folder / label / f"{issue.number}.txt"
        output_file.write_text(f"{issue.title}\n{issue.body}")


if __name__ == "__main__":
    fire.Fire(get_data)
```

</details>

<details>
<summary>Create `dvc.yaml`</summary>

```yaml
stages:
  get-data:
    cmd: python src/get_data.py
      --output_folder ${data_folder}
    deps:
      - src/get_data.py
    params:
      - repo
      - labels
      - state
      - since
      - until
    outs:
      - ${data_folder}
  
  data-metrics:
    cmd: python src/data_metrics.py
      --input_folder ${data_folder}
      --output_metrics_file ${metrics_file}
    deps:
      - src/data_metrics.py
      - ${data_folder}
    metrics:
      - ${metrics_file}:
          cache: false
```

</details>

---
