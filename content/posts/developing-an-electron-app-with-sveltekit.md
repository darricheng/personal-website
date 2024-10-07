+++
title = "Developing an Electron app with SvelteKit"
date = 2024-10-07T18:51:48+08:00
draft = false
description = "A walkthrough from my experience for getting SvelteKit working with Electron"
slug = "developing-an-electron-app-with-sveltekit"
tags = ["svelte", "sveltekit", "electron"]
+++

I chose [Electron](https://www.electronjs.org/) to create an invoicing app for my partner because I wanted to store customer data locally so that I didn't have to worry about doing authentication to protect the data when trying to access it over the internet.

Initially, I used [Tauri](https://tauri.app/) because I'm keen to become more proficient with the Rust programming language. With Tauri, I could easily use my current favoured frontend technology Svelte (more specifically using SvelteKit) as they provide [simple setup instructions](https://tauri.app/v1/guides/getting-started/setup/sveltekit) for using the two together.

However, due to some limitations, I switched over to Electron halfway through development of the invoicing app. That meant porting whatever I had already done in Rust to Electron's main process, and getting my SvelteKit frontend working with Electron. The former was straightforward, since it simply involved converting the logic written in Rust to TypeScript. There was also the dealing with differences between how Electron and Tauri work, but my app was simple enough that it didn't pose significant trouble.

As per the title, I faced significant difficulty porting my existing SvelteKit portion of the app over to work with Electron. It didn't help that there weren't many others who wanted to do the same as well (or they successfully did but I couldn't find their articles or repositories online).

By the end, I hope to shed some light on how I integrated Electron and SvelteKit together, and also explore a little on why my setup works.

## Working setup

I'll cut to the chase by first sharing the final working setup that I arrived at. Link to the repository [here](https://github.com/darricheng/invoicing-app).

I have two main folders, one for Electron and one for SvelteKit.

```
my-project/
├ electron-app/
└ svelte-app/
```

I use [electron-vite](https://electron-vite.org/) to scaffold my electron-app folder, and a fresh [SvelteKit](https://kit.svelte.dev/docs/creating-a-project) project for my svelte-app folder.

### Tools used

This is the list of tools that I use in my working setup, in no particular order:

- [electron-vite](https://electron-vite.org/)
- [SvelteKit](https://kit.svelte.dev/)
- [pnpm](https://pnpm.io/)
- [fish shell](https://fishshell.com/)
- [just](https://github.com/casey/just)
- [mprocs](https://github.com/pvolok/mprocs)
- [trash-cli](https://github.com/andreafrancia/trash-cli)
- [sd](https://github.com/chmln/sd)

### Local Development

We'll first take a look at how I set up the two sub-projects to work with each other in development. This section only deals with getting your project set up so that you can work on your app's features. Building the app binary will come in the next section.

#### Setting up SvelteKit

The main thing here is to change SvelteKit to use client-side-rendered (CSR) pages. I do so for the entire app by adding the following lines to the `+layout.ts` file at the root of the `routes/` directory (see SvelteKit's [routing](https://kit.svelte.dev/docs/routing) and [page options](https://kit.svelte.dev/docs/page-options) docs if you're unfamiliar).

```typescript
export const csr = true;
export const prerender = false;
export const ssr = false;
```

I think prerender should work as well, though that is not what I chose for my project. In hindsight, I would like to try using prerender instead of CSR, because rendering the UI from HTML directly should be faster than rendering it with JavaScript.

#### Setting up Electron

The Electron side of the app is pretty much a vanilla repo from [electron-vite](https://electron-vite.org/guide/), except with the renderer stuff deleted. That means the `src/renderer/` directory and the renderer config in `electron.vite.config.ts` is deleted. We'll instead be providing the renderer files separately from the SvelteKit sub-project.

#### Linking the two together

We have to ensure the electron app and SvelteKit server are listening to the same port on localhost. By default, this is 5173 for SvelteKit because it uses [Vite](https://vitejs.dev/config/server-options.html#server-port). In the electron main process, we match it by calling `new BrowserWindow({...options}).loadURL('http://localhost:5173')` when it is running in a dev environment, so that it loads the content from the SvelteKit server.

We can then develop the UI like any other SvelteKit app, with features like Hot Module Replacement (HMR) available out of the box. You can also configure SvelteKit to have [Svelte inspector](https://github.com/sveltejs/vite-plugin-svelte/blob/main/docs/inspector.md) active if you like.

I also created a `shared-types/` directory at the project root to store shared code, which in the case of this project was just mainly common types for data that both Electron and SvelteKit would need. For larger projects, this could definitely also be used for sharing other code, such as utility functions.

You'll have to do some tinkering with `tsconfig.json` to get both sub-projects working together with code in the shared directory. This is an area where (at the time of writing) I have limited knowledge about still. Using [TypeScript's project references feature](https://www.typescriptlang.org/docs/handbook/project-references.html), I added an empty `tsconfig.json` file in the shared code directory, then added the following to the top-level of my sub-project directories' `tsconfig.json` file.

```json
"references": [
	{
		"path": "../shared-types"
	}
]
```

I can't say for sure that the TypeScript config setup I have is the best for my desired outcome, but it worked sufficiently well for what I needed.

To start both applications together, I make use of [mprocs](https://github.com/pvolok/mprocs). I also use a [justfile](https://github.com/casey/just) to shorten the commands that I always type for development. The justfile will see much greater use when building the app too.

My mprocs config is as follows (change your package manager accordingly):

```yaml
procs:
	sveltekit:
		shell: "pnpm run dev"
		cwd: "./svelte-app/"
		stop: "SIGINT"
	electron:
		shell: "pnpm run dev"
		cwd: "./electron-app/"
		stop: "SIGTERM"
```

My basic justfile for development:

```just
default:
    just --list --unsorted

dev: check-port
    mprocs --config mprocs.dev.yaml

# exit if port 5173 is already in use
check-port:
	#!/usr/bin/env bash
	# https://just.systems/man/en/chapter_44.html#safer-bash-shebang-recipes
	set -euxo pipefail
	# lsof returns nothing if the port is not in use,
	# which is false so the script withon doesn't run
	if lsof -Pi :5173 -sTCP:LISTEN -t >/dev/null ; then
		echo "ERROR: Port 5173 is already in use"
		exit 1
	fi
```

Running `just dev` will run the `check-port` command first, then run the `dev` command. So it will check whether port 5173 is already in use, and if it's available, run `mprocs`.

The `check-port` portion is completely optional. Frankly, I never ran into that issue at all while working on my app, but it's there in case I have another project server running also using port 5173 that I might have forgotten to stop.

There will be more to the justfile in the next section, as I use it extensively to manage the scripts for building the entire application.

Note that any scripts in my justfile are based on fish shell's syntax because that's the shell that I use. You should adjust the scripts accordingly based on the shell that you use.

### Building the app

Building the final app is a little more involved than setting it up for development, as it involves combining files from both sub-projects to produce the app binary. As mentioned above, I use a justfile to manage this entire process.

Here's my [full justfile](https://github.com/darricheng/invoicing-app/blob/main/justfile) for reference. I will be referring back to it in the subsequent sections.

```just
default:
	just --list --unsorted

dev: check-port
	mprocs --config mprocs.dev.yaml

livedev: check-port
	mprocs --config mprocs.livedev.yaml

build: full-svelte trash-dist
	cd electron-app && pnpm run build:mac

test-build: full-svelte trash-dist && open
	cd electron-app && pnpm run test-build
	cd electron-app/dist && unzip 'Invoicing App-1.0.0-arm64-mac.zip'

full-svelte: build-svelte prep-svelte move-svelte

build-svelte:
	cd svelte-app && pnpm run build

prep-svelte:
	# change paths to be relative to current directory so that electron
	# can find the files (as it uses the file:// protocol)
	# see: https://stackoverflow.com/a/54481688
	sd -F '/_app' './_app' svelte-app/build/index.html

move-svelte:
	#!/usr/bin/env bash
	set -euxo pipefail # https://just.systems/man/en/chapter_44.html#safer-bash-shebang-recipes
	if [ -d "electron-app/out/renderer/" ]; then
		trash electron-app/out/renderer/
	fi
	mkdir -p electron-app/out/renderer
	cp -R svelte-app/build/* electron-app/out/renderer

trash-dist:
	#!/usr/bin/env bash
	set -euxo pipefail # https://just.systems/man/en/chapter_44.html#safer-bash-shebang-recipes
	if [ -d "electron-app/dist/" ]; then
		trash electron-app/dist/
	fi

open:
	open electron-app/dist/Invoicing\ App.app

# exit if port 5173 is already in use
check-port:
	#!/usr/bin/env bash
	set -euxo pipefail # https://just.systems/man/en/chapter_44.html#safer-bash-shebang-recipes
	# lsof returns nothing if the port is not in use, which is false so the script withon doesn't run
	if lsof -Pi :5173 -sTCP:LISTEN -t >/dev/null ; then
		echo "ERROR: Port 5173 is already in use"
		exit 1
	fi
```

#### Preparing the renderer files from SvelteKit

To build the final app, we first have to build our SvelteKit app into a set of files that can be used by electron-vite to build the final app binary. In our case, that means ending up with a set of files that has a single entry point called `index.html`, with all other necessary files loaded from there.

We will focus on the `full-svelte` command, as that is responsible for preparing our SvelteKit files. Running `just full-svelte` will run the following commands in order: `build-svelte`, `prep-svelte`, then `move-svelte`. As their names suggest, we will build the Svelte files, prep it for building with Electron, and move it to the Electron directory.

1. `build-svelte`: Straightforward command, simply runs SvelteKit's build command. The default config places the output in a `build/` directory within the svelte sub-project directory. Since we set up the app to use Client-side Rendering, the output should be a single HTML file with one or more JavaScript files.
2. `prep-svelte`: We need to modify the built files so that they work with Electron. Specifically, we change all the paths in `index.html` that try to load files from `/_app` to `./_app`. I do so with a CLI tool called [sd](https://github.com/chmln/sd). There might be a way to get the Svelte build tooling to output the desired paths directly, but I did not explore that option.
3. `move-svelte`: The last step for prepping SvelteKit's files. We move the built and modified files to the correct location in the Electron sub-project directory so that we can build the final app binary.

#### Building the app binary

With the Svelte files prepped and moved to the correct location, we can run the build command in the Electron sub-project to build the final app binary. I do so with `just build`.

This command first preps the Svelte files by running the `full-svelte` command as I described in the previous section. Then, it runs the `trash-dist` command, which checks whether a previous build exists in the electron sub-project's `dist/` directory. If `dist/` exists, the command trashes the directory using [trash-cli](https://github.com/andreafrancia/trash-cli). You can replace this with `rm`.

Lastly, the `just build` command runs the build command that comes with the default initialisation of an electron-vite project. Specifically for my purposes, I directly call the command to build the app for MacOS because that's the OS I use.

I do have a `just test-build` command that I use more in development when testing the final app binaries to speed up the iteration process. The `pnpm run test-build` command is the same as the `pnpm run build` command, but it skips the type-checking step. I also set it up to immediately open the newly built binary for me, so that I don't have to go to the specific directory every time just to extract and open the binary to test the built app.

## Understanding why my setup works

In the rest of the article, I will cover the things I learnt about how and why my above setup works. If you're here just to get a working setup, then that is all I have for you, otherwise do read on!

### Local Development

Electron is able to function as a web browser by [loading URLs directly](https://www.electronjs.org/docs/latest/api/browser-window#winloadurlurl-options), though note that this is not recommended due to [security risks](https://www.electronjs.org/docs/latest/tutorial/security). Still, I bring up this functionality because that is how we will get our SvelteKit project working with Electron to give us a nice development experience.

When developing, we use `BrowserWindow.loadUrl('http://localhost:5173')` after starting up the vite server in SvelteKit to load our SvelteKit app in Electron. Contrast this with using `BrowserWindow.loadFile(join(__dirname, '../renderer/index.html'));` in production builds to load the files built by SvelteKit from the file system.

Because vite serves the files that we need when developing, we also get features like Hot Module Replacement when developing our Electron app. This is possible because vite is the one that re-serves files to the browser, which in this case is Electron, so that we see updated changes as we edit our code.

### Building the app

As mentioned in the Working Setup section, building the app is more involved than getting it ready for development. There was more I had to figure out here to build a final app binary for my project.

#### How electron-vite builds the app binary

There are two steps to the build process:

1. Run `electron-vite build` to build our files into a format that can be used by the final binary.
2. Run `electron-builder --mac`, replacing `mac` with the relevant target platform, to build the final app binary.

In step 1, electron-vite would usually build our main, preload, and renderer files, then [place them in the `out/` directory](https://electron-vite.org/guide/build) ready for step 2. However, since we removed the renderer files from our electron sub-project, there won't be any built renderer files, so we have to place the files in the correct directory ourselves to prepare for step 2. This is where the `just full-svelte` command comes into play.

#### Preparing Svelte files for the Electron build step

```just
full-svelte: build-svelte prep-svelte move-svelte

build-svelte:
	cd svelte-app && pnpm run build

prep-svelte:
	# change paths to be relative to current directory so that electron
	# can find the files (as it uses the file:// protocol)
	# see: https://stackoverflow.com/a/54481688
	sd -F '/_app' './_app' svelte-app/build/index.html

move-svelte:
	#!/usr/bin/env bash
	set -euxo pipefail # https://just.systems/man/en/chapter_44.html#safer-bash-shebang-recipes
	if [ -d "electron-app/out/renderer/" ]; then
		trash electron-app/out/renderer/
	fi
	mkdir -p electron-app/out/renderer
	cp -R svelte-app/build/* electron-app/out/renderer
```

The commands that I will focus on are prep-svelte and move-svelte, since build-svelte is just building the SvelteKit app into a set of static HTML and JavaScript files using the build command provided by a standard SvelteKit project.

For `prep-svelte`, we edit the `index.html` file produced by the SvelteKit build process so that our final binary works. By default, SvelteKit places the built files in the `_app/` directory located in the same parent directory as `index.html`.

```
some-electron-directory/
├ index.html
└ _app/
```

As you can see from the command, I change the path of content loaded from the `_app/` directory using [sd](https://github.com/chmln/sd) to include the base path of `index.html` by adding a `.` so that `/_app/path/to/file` becomes `./_app/path/to/file`.

This change is necessary because Electron loads files using the [file protocol](https://en.wikipedia.org/wiki/File_URI_scheme). With the path being `/_app/path/to/file`, Electron will attempt to load the files from the root (`/`) directory, which of course won't have our files. Adding the `.` in front tells Electron to load the files with the current directory of `index.html` as the base directory, allowing it to correctly find our `_app/` directory and load the files needed for our app to function.

For `move-svelte`, it's simply about moving the files, including the `index.html` that we just edited in `prep-svelte`, to the right place in the electron sub-project. As mentioned in the previous section about building the final app binary, this is `electron-app/out/renderer/`. To make sure we have the most updated files, we do a check whether a directory already exists at that path. If it does, we delete it, then create a new directory and copy our prepped Svelte files into it.

Once we have the files in the right place, running the appropriate build command should build a working final app binary. Using my setup, running `just build` will do everything in order, from preparing the Svelte files accordingly, to running the build command for MacOS.

And that is it! Hopefully, by getting Electron and SvelteKit working together, I was able to shed some light on how they each work individually as well. Go forth now and use SvelteKit for your Electron app!
