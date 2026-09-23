# Agent Team

Agent Team is an early-preview command-line tool for setting up a configurable
team of AI agents on your own machine. The default team has
three roles—Architect, Developer, and Tester—but each role, instruction file,
and workflow assignment is configuration-driven.

The local deployment consists of:

- A Supervisor that coordinates work, policy, durable state, and messaging.
- One isolated Docker container per agent, with Codex CLI and Git included.
- A private Zulip channel, a Supervisor bot, and one bot identity per agent.
- A user-owned JSON definition and separate `AGENTS.md` instructions for every
  role.

The project is designed around GitHub repositories and issues as the work
record, with Zulip as the communication platform for people and agents.

> [!IMPORTANT]
> Agent Team is not yet a complete end-to-end agent runtime. The current public
> preview can initialize a team, provision Zulip, and create a Docker
> deployment. A supported command to start that deployment, plus GitHub and
> Codex credential setup for the running agents, has not shipped yet. Use this
> release to evaluate the configuration and provisioning workflow.

## Requirements

- Windows x64.
- [Docker Desktop](https://docs.docker.com/desktop/setup/install/windows-install/)
  running Linux containers, with Docker Compose available.
- Network access to GitHub, Docker Hub, and your Zulip server.
- An existing Zulip organization and an active human account allowed to create
  bots and private channels.
- A personal [`zuliprc` file](https://zulip.com/api/api-keys) for that account.

The released executable is self-contained, and `agent-team create` pulls the matching, digest-pinned
Supervisor and Agent Runner images.

## Install

Download `agentteam-win-x64.exe` from the
[latest release](https://github.com/stick109/agent-team-releases/releases/latest).
You can rename it to `agent-team.exe` and place it in a directory on your
`PATH`.

Alternatively, download it from PowerShell:

```powershell
Invoke-WebRequest `
  'https://github.com/stick109/agent-team-releases/releases/latest/download/agentteam-win-x64.exe' `
  -OutFile .\agent-team.exe

.\agent-team.exe version
```

The examples below assume the renamed executable is available as `agent-team`.
If it is not on `PATH`, use its full path instead.

## Quick start

### 1. Initialize a team

Choose a path that does not already exist:

```powershell
agent-team init my-team
```

Key files include:

```text
my-team/
  agent-team.json
  agents/
    architect/AGENTS.md
    developer/AGENTS.md
    tester/AGENTS.md
  .agent-team/
    secrets/
```

`init` is local-only. It does not contact Zulip, GitHub, or Docker.

### 2. Configure the team

Open `my-team/agent-team.json` and, at minimum, replace the generated values for:

- `team.id`
- `repository.slug` and `repository.default_branch`
- `communication.zuliprc_path`

The key section should resemble this:

```json
{
  "team": {
    "id": "my-product-team"
  },
  "repository": {
    "provider": "github",
    "slug": "your-organization/your-repository",
    "default_branch": "main"
  },
  "communication": {
    "provider": "zulip",
    "zuliprc_path": "~/.zuliprc"
  }
}
```

Keep the other generated fields in the real file. Edit the three generated
`AGENTS.md` files to define what each role may do and how it should work.

Treat `.zuliprc` and everything under `.agent-team/secrets/` as credentials.
Do not commit them or share them.

### 3. Provision the team

Run `create` from the parent directory:

```powershell
agent-team create my-team
```

Or run it without a directory while inside the team directory:

```powershell
Set-Location my-team
agent-team create
```

`create` first checks the local configuration, required credential file,
Docker, and release metadata, then pulls or verifies the container images. It
then:

1. Creates or reconciles the private Zulip channel, Supervisor bot, agent bots,
   and subscriptions.
2. Stores generated runtime state and bot credentials under `.agent-team/`.
3. Creates the Docker network, persistent volumes, Supervisor container, and one
   Agent Runner container per configured role.
4. Leaves all containers stopped.

Rerunning `create` checks and reconciles the same installation. It does not
silently replace a deployment whose locked version or topology differs.

## Manage Zulip

```powershell
# Show provisioning and connection state.
agent-team zulip status my-team

# Reconcile missing or drifted Zulip resources.
agent-team zulip repair my-team

# Replace one managed bot key.
agent-team zulip rotate my-team --bot developer

# Allow a human to exchange direct messages with managed bots.
agent-team zulip allow-user my-team person@example.com

# Deactivate managed bots and disconnect the installation.
agent-team zulip disconnect my-team
```

Commands that need the privileged human credential read the configured
`zuliprc` file. You can provide a one-time replacement with `--zuliprc PATH`.

`allow-user` asks you to type `yes`. `disconnect` asks you to type `disconnect`.
Add `--archive-channel` only when you also want to archive the managed channel;
that action asks you to type `archive` separately.

## Remove the Docker deployment

```powershell
agent-team destroy my-team
```

`destroy` asks for confirmation, then permanently removes the team's Docker
containers, network, Supervisor data, agent state, and agent workspaces. It
preserves `agent-team.json`, role instructions, local provisioning files,
downloaded images, and all remote Zulip resources.

To remove the managed Zulip bots separately, use `agent-team zulip disconnect`.
Use `agent-team destroy my-team -f` only when you intentionally want to skip the
Docker-deletion confirmation.

## Command reference

```text
agent-team init <directory>
agent-team create [directory] [--zuliprc PATH]
agent-team destroy [directory] [-f]
agent-team zulip status [directory]
agent-team zulip repair [directory] [--zuliprc PATH]
agent-team zulip rotate [directory] --bot ROLE [--zuliprc PATH]
agent-team zulip allow-user [directory] <email-or-user-id> [--zuliprc PATH]
agent-team zulip disconnect [directory] [--zuliprc PATH] [--archive-channel]
agent-team version
```

Installing a newer executable does not upgrade an existing team deployment.
An automated deployment-upgrade command is not available in the current
preview.

## Releases and support

Current release builds pair the Windows executable with an
`agent-team-release-manifest.json` that identifies the source revision and pins
the matching container images by immutable digest. When both files are present,
download them from the same release when inspecting or archiving a version.

Report release or documentation problems in
[GitHub Issues](https://github.com/stick109/agent-team-releases/issues).
