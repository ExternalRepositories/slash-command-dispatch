# Slash Command Dispatch
[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-Slash%20Command%20Dispatch-blue.svg?colorA=24292e&colorB=0366d6&style=flat&longCache=true&logo=data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAA4AAAAOCAYAAAAfSC3RAAAABHNCSVQICAgIfAhkiAAAAAlwSFlzAAAM6wAADOsB5dZE0gAAABl0RVh0U29mdHdhcmUAd3d3Lmlua3NjYXBlLm9yZ5vuPBoAAAERSURBVCiRhZG/SsMxFEZPfsVJ61jbxaF0cRQRcRJ9hlYn30IHN/+9iquDCOIsblIrOjqKgy5aKoJQj4O3EEtbPwhJbr6Te28CmdSKeqzeqr0YbfVIrTBKakvtOl5dtTkK+v4HfA9PEyBFCY9AGVgCBLaBp1jPAyfAJ/AAdIEG0dNAiyP7+K1qIfMdonZic6+WJoBJvQlvuwDqcXadUuqPA1NKAlexbRTAIMvMOCjTbMwl1LtI/6KWJ5Q6rT6Ht1MA58AX8Apcqqt5r2qhrgAXQC3CZ6i1+KMd9TRu3MvA3aH/fFPnBodb6oe6HM8+lYHrGdRXW8M9bMZtPXUji69lmf5Cmamq7quNLFZXD9Rq7v0Bpc1o/tp0fisAAAAASUVORK5CYII=)](https://github.com/marketplace/actions/slash-command-dispatch)

A GitHub action that facilitates "ChatOps" by creating repository dispatch events for slash commands.

### How does it work?

The action runs in `on: issue_comment` workflows and checks comments for slash commands.
When a valid command is found it creates a repository dispatch event that includes a payload containing full details of the command and its context.

### Why repository dispatch?

"ChatOps" with slash commands can work in a basic way by parsing the commands during `issue_comment` events and immediately processing the command.
In repositories with a lot of activity, the workflow queue will get backed up very quickly if it is trying to handle new comments for commands, **and** process the commands themselves.

Dispatching commands to be processed elsewhere keeps the workflow queue moving quickly. It essentially allows you to run multiple workflow queues in parallel.

### Advantages of slash-command-dispatch

- Easy configuration of "ChatOps" slash commands
- Separating the queue of `issue_comment` events from the queue of commands to process keeps it fast moving
- Users receive faster feedback that commands have been seen and are waiting to be processed
- The ability to handle processing workloads in multiple repositories in parallel
- Long running workloads can be processed in a repository workflow queue of their own
- Even if commands are processed in the same repository, separation of comment parsing and command processing logic allows easier management of "ChatOps" use cases

### Demo



## Usage

### Basic configuration

```yml
name: Slash Command Dispatch
on:
  issue_comment:
    types: [created]
jobs:
  slashCommandDispatch:
    runs-on: ubuntu-latest
    steps:
      - name: Slash Command Dispatch
        uses: peter-evans/slash-command-dispatch@v1
        with:
          token: ${{ secrets.REPO_ACCESS_TOKEN }}
          commands: rebase, integration-test, create-ticket
          repository: peter-evans/slash-command-dispatch-processor
```

### Action inputs

| Name | Description | Default |
| --- | --- | --- |
| `token` | (**required**) A `repo` scoped [PAT](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line). | |
| `reaction-token` | `GITHUB_TOKEN` or a `repo` scoped [PAT](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line). | |
| `reactions` | Add reactions to comments containing commands. | `true` |
| `commands` | (**required**) A comma separated list of commands to dispatch. | |
| `permission` | The permission level required by the user to dispatch commands. (`none`, `read`, `write`, `admin`) | `write` |
| `issue-type` | The issue type required for commands. (`issue`, `pull-request`, `both`) | `both` |
| `allow-edits` | Allow edited comments to trigger command dispatches. | `false` |
| `repository` | The full name of the repository to send the dispatch events. | Current repository |
| `event-type-suffix` | The repository dispatch event type suffix for the commands. | `-command` |
| `config` | JSON configuration for commands. | |
| `config-from-file` | JSON configuration from a file for commands. | |

### Handling dispatched commands

Repository dispatch events have a `type` to distinguish between events. The `type` set by the action is a combination of the slash command and `event-type-suffix`. The `event-type-suffix` input defaults to `-command`.

For example, if your slash command is `integration-test`, the event type will be `integration-test-command`.

```yml
on:
  repository_dispatch:
    types: [integration-test-command]
```

### What is the reaction-token?

If you don't specify a token for `reaction-token` it will use the [PAT](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line) supplied via `token`.
This means that reactions to comments will appear to be made by the user account associated with the PAT. If you prefer to have the @github-actions bot user react to comments you can set `reaction-token` to `GITHUB_TOKEN`.

```yml
      - name: Slash Command Dispatch
        uses: peter-evans/slash-command-dispatch@v1
        with:
          token: ${{ secrets.REPO_ACCESS_TOKEN }}
          reaction-token: ${{ secrets.GITHUB_TOKEN }}
          commands: rebase, integration-test, create-ticket
```

### Advanced configuration

Using JSON configuration allows the options for each command to be customised.

Note that it's recommended to write the JSON configuration directly in the workflow rather than use a file. Using the `config-from-file` input will be slightly slower due to requiring the repository to be checked out with `actions/checkout` so the file can be accessed.

```yml
name: Slash Command Dispatch
on:
  issue_comment:
    types: [created]
jobs:
  slashCommandDispatch:
    runs-on: ubuntu-latest
    steps:
      - name: Slash Command Dispatch
        uses: peter-evans/slash-command-dispatch@v1
        with:
          token: ${{ secrets.REPO_ACCESS_TOKEN }}
          reaction-token: ${{ secrets.GITHUB_TOKEN }}
          config: >
            [
              {
                "command": "rebase",
                "permission": "admin",
                "issue_type": "pull-request",
                "repository": "peter-evans/slash-command-dispatch-processor"
              },
              {
                "command": "integration-test",
                "permission": "write",
                "issue_type": "both",
                "repository": "peter-evans/slash-command-dispatch-processor"
              },
              {
                "command": "create-ticket",
                "permission": "write",
                "issue_type": "issue",
                "allow_edits": true,
                "event_type_suffix": "-cmd"
              }
            ]
```

An example using the `config-from-file` input to set JSON configuration.
Note that `actions/checkout` is required to access the file.

```yml
name: Slash Command Dispatch
on:
  issue_comment:
    types: [created]
jobs:
  slashCommandDispatch:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v1
      - name: Slash Command Dispatch
        uses: peter-evans/slash-command-dispatch@v1
        with:
          token: ${{ secrets.REPO_ACCESS_TOKEN }}
          config-from-file: .github/slash-command-dispatch.json
```

## License

[MIT](LICENSE)
