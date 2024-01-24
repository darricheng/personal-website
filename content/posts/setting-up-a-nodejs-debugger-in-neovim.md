+++
title = "Setting up NodeJS debugging in Neovim"
date = 2023-12-21T20:18:57+08:00
draft = false
description = "How to set up a debugger for NodeJS in Neovim"
slug = "setting-up-nodejs-debugging-in-neovim"
tags = ["javascript", "typescript", "debugging", "nodejs"]
+++

## Introduction

Setting up a debugger was not easy or straightforward for me. Nothing works out of the box as compared to common IDEs, such as VS Code, so there was plenty of learning needed to get me to my current point. I wrote this piece in hopes that it might help someone else understand a little bit more about debuggers through trying to set one up for their own Neovim configurations too.

I still have plenty that I don't know about debuggers and the Debug Adapter Protocol. For example, at the point of writing this I still wasn't able to get the debugger set up and running properly to debug a NestJS app running in a Docker container (oddly specific, but it was a use case I required). But I'm happy with where I arrived at so far and want to share my learnings.

## Debug Adapter Protocol

The first thing that I needed to gain some understanding about is the [Debug Adapter Protocol (DAP)](https://microsoft.github.io/debug-adapter-protocol/). DAP is a standard created by Microsoft that attempts to unify the different kinds debuggers so that we can use them all from a single common interface. This common interface can be implemented independent of the protocol, so that we can get the same debugging experience across editors and IDEs.

There are four main parts to the protocol:

1. DAP client: what we interact with when we use a debugger in an IDE
2. Debug Adapter: a layer of abstraction above the debugger that implements the protocol for the debugger in question
3. Debugger: the actual application doing the debugging work
4. Debuggee: the program we are trying to debug

For my setup in Neovim, I use [nvim-dap](https://github.com/mfussenegger/nvim-dap) as my DAP client. I also use [nvim-dap-ui](https://github.com/rcarriga/nvim-dap-ui) that sets up a nice user interface to do debugging. Another option that I'm aware of is [Vimspector](https://github.com/puremourning/vimspector), but I have not tried it because the recommendations for Neovim generally point to the options I have mentioned.

### Installing the Debug Adapter

I use [mason.nvim](https://github.com/williamboman/mason.nvim/) to manage the installations of my language tools, including debug adapters. For NodeJS, the debug adapter I use is called `js-debug-adapter`, which is really just [vscode-js-debug](https://github.com/microsoft/vscode-js-debug), the debug adapter for JavaScript that comes with VS Code.

## DAP Configurations

There are two main things to configure.

1. Adapter: How the DAP client (nvim-dap) should start the debugger.
2. Debugger: How the debugger should connect to the debuggee.

### Adapter Config

We use the dap adapters config object to configure how our DAP client should start the debugger. There are three types of debuggers that exist:

- Executable
- Server
- Pipe

Each has their own setup configuration options, but the main idea is the same. At this step, we need to tell nvim-dap how to go about starting the debugger that we want, so that we can work with the debugger correctly.

You'll have to look at the specific debug adapter implementation for which configuration type to use. For example, [`vscode-js-debug`](https://github.com/microsoft/vscode-js-debug) and [`codelldb`](https://github.com/vadimcn/codelldb) are both server types, so we will have to use the server-specific configuration options to set them up for use with our DAP client.

To set up the debugger for NodeJS, I use the following configurations passed to nvim-dap's adapters config table:

```lua
['pwa-node'] = {
	type = 'server',
	host = '::1',
	port = '${port}',
	executable = {
		command = 'js-debug-adapter',
		args = {
			'${port}',
		},
	},
},
```

Here's a brief description of each option:

- `['pwa-node']`: The name of the debug adapter when used in configurations is called `pwa-node` (see [stackoverflow](https://stackoverflow.com/a/64261636) and [GitHub issue](https://github.com/microsoft/vscode/issues/151910) for context).
- `type = 'server'`: The NodeJS debug adapter runs as a server.
- `host = '::1'`: We run the debug adapter locally, so that means it'll be listening on localhost for connections.
- `port`: Port that our dap client will use to communicate with the dap adapter, in this case `${port}` means nvim-dap will automatically assign a random open port.
- `executable`: Options here determine how nvim-dap will start the dap adapter.
  - `command`: The command used to start the debug adapter.
  - `args`: Any additional arguments that are required with the command, in this case we pass the random port assigned by nvim-dap so that it can correctly connect to the debugger.

Detailed descriptions of all the available options and what each option does can be found in the nvim-dap help docs (`:h dap.txt`). There you'll also find the available options for the other adapter types (executable and pipe).

### Debugger Config

The debugger config determines how the debugger should interact with the debuggee. We can have different configurations to support the many different ways a program can be debugged.

Below is a basic example of what I use for launching and debugging a locally running NodeJS program.

```lua
dap.configurations['typescript'] = {
	{
		type = 'pwa-node',
		request = 'launch',
		name = 'Launch file',
		program = '${file}',
		cwd = '${workspaceFolder}',
	},
	{
		-- another set of configs here
	}
}
```

The first step is to define the languages for which the configurations can be used. In this case, the configurations will be available for typescript files.

For each set of configurations, there are three required options for all debuggers that you configure with nvim-dap:

- type: The debug adapter to use.
- request: Can be either `attach` or `launch`.
- name: A unique string used to identify the configuration.

The configuration object also accepts an arbitrary number of additional fields that will be used to configure the debug adapter in question.

The available configuration options for the NodeJS debug adapter can be found in the [options document](https://github.com/microsoft/vscode-js-debug/blob/main/OPTIONS.md) of the debugger repository.

_Note: Each debug adapter has its own unique set of options, so you'll have to refer to the specific adapter's documentation. For example, [codelldb has its own set of configuration options](https://github.com/vadimcn/codelldb/blob/master/MANUAL.md) that is different from vscode-js-debug._

I'll briefly go over the two different request types and how to use each, with some settings for using each config.

#### Launch

With this configuration, the debug adapter is responsible for launching the application that we are trying to debug. That means we will restart the application every time we start a new debugging session.

Refer to the [code snippet in the debugger config section](#debugger-config) for a basic launch configuration for NodeJS. Below describes each option.

- `type`: The debug adapter that we are using, which is `pwa-node`.
- `request`: The debugger will launch the application for debugging.
- `name`: An identifier for this debug configuration.
- `program`: The program that the debugger will launch, which we have set as the file open in the active Neovim buffer.
- `cwd`: Path to the working directory of the program being debugged, which we have set as the current working directory of Neovim.

#### Attach

The adapter attempts to attach to an application that is already running. The app must have a configured debug port for the debugger to attach to upon startup. In other words, the app must be listening on a port (e.g. 9229) so that the debugger can initiate a connection with the running application for debugging.

Below is a basic attach configuration for NodeJS.

```lua
{
	type = 'pwa-node',
	request = 'attach',
	name = 'Attach to Node app',
	address = 'localhost',
	port = 9229,
	cwd = '${workspaceFolder}',
	restart = true,
},
```

Refer to the [launch configuration](#launch) for the descriptions of type, request, name, and cwd.

- `address`: The address at which your application is running.
- `port`: The port of the application that is exposed for debugging. The default for NodeJS is 9229.
- `restart`: Try to reconnect to the application if we lose connection.

## Starting A Debug Session

With the configurations in place, starting a debug session is just a matter of setting breakpoints in your code, then starting the debugger.

If you created multiple configurations for the language of the open file, nvim-dap will show a menu for you to select the specific configuration that you want to use.

And that is how you set up a debugger in Neovim specifically for NodeJS! Setting other debuggers should just be a matter of adapting what you have for NodeJS for the other language and debuggers.

I hope that this was useful for you, and that you learnt something in the process as well.
