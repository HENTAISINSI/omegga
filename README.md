# Omegga

Similar to [n42k's brikkit](https://github.com/n42k/brikkit), wraps brickadia's server console to provide interactivity via plugins.

## Install

You can run omegga in the Windows Subsystem for Linux or on an actual linux install.

Omegga depends on:

  * Node v12+ ([windows](https://nodejs.org/en/download/), [ubuntu/deb](https://github.com/nodesource/distributions/blob/master/README.md#installation-instructions))
  * one of:
    * the **brickadia linux launcher** which is not available publicly at the moment.
    * `apt install wget tar` (to download a4 binary)

Omegga is installed as a global npm package

    npm i -g omegga

Alternatively, you can use a development/local omegga

    # clone omegga
    git clone https://github.com/brickadia-community/omegga.git && cd omegga

    # point development omegga to global npm bin
    npm link

## Running

To start a server, simply type the following in a linux shell after install:

    omegga

Omegga will prompt for credentials as necessary and only stores the auth tokens brickadia generates on login. **Omegga does not store your password**

## Screenshots

![Generic omegga screenshot](https://i.imgur.com/AqJF2T0.png)

# Planned Features

  * [ ] web interface
    * [ ] reload plugins
    * [ ] enable/disable plugins live
    * [ ] manage plugins config, perhaps by iframe
    * [ ] chat with players
    * [ ] view recent console logs
    * [ ] view server status
  * [x] terminal interface
    * [x] reload plugins
    * [x] chat with players
    * [x] view recent console logs
    * [x] view server status
  * [ ] metrics
    * [ ] bricks over time charts
    * [ ] player online time tracking
    * [ ] chat logs
    * [ ] chats/hour tracking
  * [x] plugins in other languages via JSON RPC over stdio
    * [ ] LogWrangler impl for other languages
    * [x] events sent JSON RPC
  * [ ] sandboxed node plugins (more secure, more stable)
    * [x] running in own thread (worker)
    * [x] running in own vm
    * [x] can `require`
    * [x] partial omegga spec (events, some features)
    * [x] full omegga spec
    * [ ] _good_ access restrictions (ask user for permission)
  * [ ] plugin installation by `omegga install gh@user/repo`
  * [ ] plugin updates by `omegga update`
  * [ ] server config bundling (making it easier to transfer configs)
    * [ ] omegga.server.json
      * [ ] list of installed omegga plugins, versions, and download urls
      * [ ] list of roles, bans, role assignments

# Plugins

Plugins are located in the `plugins` directory in an omegga config folder

Plugins are most easily developed in Javascript at the moment using the Node VM Plugins and Node Plugins. You can use JSON RPC Plugins to write plugins in other languages.

## Plugin Structure

All plugins are located in a `plugins` directory where you are running Omegga:

* `plugins/myPlugin` - plugin folder (required)
* `plugins/myPlugin/doc.json` - plugin information (required)
* `plugins/myPlugin/disable.omegga` - empty file only present if the plugin should be disabled (optional)

Every plugin requires a `doc.json` file to document which briefly describes the plugin and its commands. In the future.

### `doc.json` (example)

```json
{
  "name": "My Plugin",
  "description": "Example Plugin",
  "author": "cake",
  "commands": [
    {
      "name": "!ping",
      "description": "sends a pong to the sender",
      "example": "!ping foo bar",
      "args": [
        {
          "name": "args",
          "description": "random filler arguments",
          "required": false
        }
      ]
    },
    {
      "name": "!pos",
      "description": "announces player position",
      "example": "!pos",
      "args": []
    }
  ]
}
```

## Node VM Plugins

Node VM Plugins are what you should be using. They are run inside a VM inside a Worker. This means when they crash, they do not crash the whole server and they can in the future have locked down permissions (disable filesystem access, etc).

These plugins receive a "proxy" reference to `omegga` and have limited reach for what they can touch.

### Globals

* `OMEGGA_UTIL` - access to the `src/util/index.js` module
* `Omegga` - access to the "proxy" omegga
* `console.log` - and other variants (`console.error`, `console.info`) print specialized output to console

### Folder Structure

In a `plugins` directory create the following folder structure:

* `plugins/myPlugin` - plugin folder (required)
* `plugins/myPlugin/omegga.plugin.js` - js plugin main file (required)
* `plugins/myPlugin/doc.json`
* `plugins/myPlugin/access.json` - plugin access information (required, but doesn't have to have anything right now). this will contain what things the vm will need to access

### `access.json` (examples)

Access to any builtin modules (`fs`, `path`, etc)
```json
["*"]
```

Access to nothing - only the code in the `omegga.plugin.js`
```json
[]
```

Access to only `fs`, (`const fs = require('fs');`)
```json
["fs"]
```

### `omegga.plugin.js` (example)

```javascript
class PluginName {
  // the constructor also contains an omegga if you don't want to use the global one
  constructor(omegga) {
    this.omegga = omegga;
    console.info('constructed my plugin!');
  }

  init() {
    Omegga
      .on('chatcmd:ping', (name, ...args) => {
        Omegga.broadcast(`pong @ ${name} + ${args.length} args`);
      })
      .on('chatcmd:pos', async name => {
        const [x, y, z] = await Omegga.getPlayer(name).getPosition();
        Omegga.broadcast(`<b>${name}</> is at ${x} ${y} ${z}`);
      });
  }

  stop() {
    // any remove events are not necessary because the VM removes the code
  }
}

module.exports = PluginName;
```


## Node Plugins

Node plugins are effectively `require`'d into omegga. They have the potential to crash the entire service through uncaught exceptions and also can be insecure. Develop and run these at your own risk - your server stability may suffer.

These plugins receive a direct reference to the `omegga` that wraps the brickadia server. As a result, they can directly modify how omegga runs.

Cleanup is important as code can still be running after the plugin is unloaded resulting in strange and undefined behavior. Make sure to run `clearInterval` and `clearTimeout`

### Globals

  * `OMEGGA_UTIL` - access to the `src/util/index.js` module

### Folder Structure

In a `plugins` directory create the following folder structure:

* `plugins/myPlugin` - plugin folder (required)
* `plugins/myPlugin/doc.json`
* `plugins/myPlugin/omegga.main.js` - js plugin main file (required)

### `omegga.main.js` (example)

```javascript
class PluginName {
  constructor(omegga) {
    this.omegga = omegga;
  }

  init() {
    this.omegga
      .on('chatcmd:ping', (name, ...args) => {
        this.omegga.broadcast(`pong @ ${name} + ${args.length} args`);
      })
      .on('chatcmd:pos', async name => {
        const [x, y, z] = await this.omegga.getPlayer(name).getPosition();
        this.omegga.broadcast(`<b>${name}</> is at ${x} ${y} ${z}`);
      });
  }

  stop() {
    this.omegga
      .removeAllListeners('chatcmd:ping')
      .removeAllListeners('chatcmd:pos');
  }
}

module.exports = PluginName;
```

## JSON RPC Plugins

JSON RPC Plugins let you use any language you desire, as long as you can run it from a single execuable file. They follow the [JSON-RPC 2.0 Specification](https://www.jsonrpc.org/specification)

The server communicates with the plugin by sending messages to `stdin` and expects responses in `stdout`. All `stderr` is printed to the console.

### Omegga Methods (You can access these)

| Method | Arguments | Description |
| ------ | --------- | ----------- |
| `log` | line (string) | Prints message to omegga console |
| `error` | line (string) | Same as `log` but with different colors |
| `info` | line (string) | Same as `log` but with different colors |
| `warn` | line (string) | Same as `log` but with different colors |
| `trace` | line (string) | Same as `log` but with different colors |
| `exec` | cmd (string) | Writes a console command to Brickadia |
| `writeln` | cmd (string) | Same as `exec` |
| `broadcast` | line (string) | Broadcasts a message to the server|
| `whisper` | {target: string, line: string} | (a5 only) Sends a message to a specific client |
| `getPlayers` | _none_ | Gets online players |
| `getRoleSetup` | _none_ | Gets server roles |
| `getBanList` | _none_ | Gets list of bans |
| `getSaves` | _none_ | Gets saves in the saves directory |
| `getSavePath` | name (string) | Gets the path to a specific save |
| `getSaveData` | _none_ | Saves the server, converts that save into a brs-js save object, returns the object |
| `clearBricks` | {target: string, quiet: bool (a5 only)} | Clears a specific player's bricks |
| `clearAllBricks` | quiet (bool, a5 only) | Clears all bricks on the server |
| `saveBricks` | name (string) | Save bricks to a save named `name` |
| `loadBricks` | {name: string, offX=0 (Number), offY=0 (Number), offY=0 (Number), quiet: bool (a5 only)} | Load bricks of save named `name` |
| `readSaveData` | name (string) | Parses save into a brs-js save object, returns the object |
| `loadSaveData` | {data: object, offX=0 (Number), offY=0 (Number), offY=0 (Number), quiet: bool (a5 only)} | Loads brs-js save data object to the server |

### Plugin Methods (You implement these)

| Method | Arguments | Description | Required |
| ------ | --------- | ----------- | -------- |
| `init` | _none_ | Returns _something_, called when plugin starts | &#9745; |
| `stop` | _none_ | Returns _something_, called when plugin is stopped | &#9745; |
| `bootstrap` | [{ object full of omegga info (`host`, `version`, etc) }] | Run when plugin is started for base data | |
| `plugin:players:raw` | [[... [player `name`, `id`, `controller`, `state`] ]] | Lists players on the server | |
| `line` | [brickadiaLog string] | A brickadia console log | |
| `start` | _none_ | Run when the brickadia server starts | |
| `host` | [{name, id}] | Run when the brickadia server detects the host | |
| `version` | ['a4' or 'a5'] | Run when the brickadia server detects the version | |
| `unauthorized` | _none_ | Run when the brickadia server fails an auth check | |
| `join` | [{name, id, state, controller}] | Run when a player joins | |
| `leave` | [{name, id, state, controller}] | Run when a player leaves | |
| `cmd:command` | [playerName, ...args] | (a5 only) Runs when a player runs a `/command args` | |
| `chatcmd:command` | [playerName, ...args] | Runs when a player runs a `!command args` | |
| `chat` | [playerName, message] | Runs when a player sends a chat message | |

### Folder Structure

In a `plugins` directory create the following folder structure:

* `plugins/myPlugin` - plugin folder (required)
* `plugins/myPlugin/doc.json`
* `plugins/myPlugin/omegga_plugin` - executable plugin file (required)

### `omegga_plugin` (example, node javascript)

```javascript
#!/usr/bin/env node

const readline = require('readline');
const { EventEmitter } = require('events');
const { JSONRPCServer, JSONRPCServerAndClient, JSONRPCClient } = require('json-rpc-2.0');

// events
const ev = new EventEmitter();

// stdio handling
const rl = readline.createInterface({input: process.stdin, output: process.stdout,terminal: false});

// rpc "server and client" for responding/receiving messages
const rpc = new JSONRPCServerAndClient(
  new JSONRPCServer(),
  // the client outputs JSON to console
  new JSONRPCClient(async blob => console.log(JSON.stringify(blob))),
);

// on stdin, pass into rpc
rl.on('line', line => {
  try {
    rpc.receiveAndSend(JSON.parse(line))
  } catch (e) {
    console.error(e);
  }
});

// regexes for matching brickadia console logs
const GENERIC_LINE_REGEX = /^(\[(?<date>\d{4}\.\d\d.\d\d-\d\d.\d\d.\d\d:\d{3})\]\[\s*(?<counter>\d+)\])?(?<generator>\w+): (?<data>.+)$/;
const LOG_LINE_REGEX = /\[(?<date>\d{4}\.\d\d.\d\d-\d\d.\d\d.\d\d:\d{3})\]\[\s*(?<counter>\d+)\](?<rest>.*)$/

ev.on('line', line => {
  const logMatch = line.match(LOG_LINE_REGEX);
  if (!logMatch) return
  const {groups: { rest }} = logMatch;
  const dataMatch = rest.match(GENERIC_LINE_REGEX);
  if (dataMatch)
    ev.emit('logData', dataMatch.groups)
  else
    ev.emit('logLine', rest);
})

// list of players
let players;

// get a player by name
const getPlayer = name => players.find(p => p.name === name);

// watch console logs for a pattern, then remove the listener
function watch(exec, pattern) {
  return new Promise(resolve => {
    function listener(line) {
      const match = line.match(pattern);
      // listener removes itself on a match
      if (match) {
        ev.off('logLine', listener);
        resolve(match.groups);
      }
    }
    // add the listener
    ev.on('logLine', listener);

    // run the console command
    rpc.notify('writeln', exec);
  })
}

// get a player's position
async function getPlayerPos(name) {
  const player = getPlayer(name);
  if (!player) return;

  // get player position from player controller
  const pawnRegExp = new RegExp(`BP_PlayerController_C .+?PersistentLevel\\.${player.controller}\.Pawn = BP_FigureV2_C'.+?:PersistentLevel.(?<pawn>BP_FigureV2_C_\\d+)'`);
  const { pawn } = await watch(`GetAll BP_PlayerController_C Pawn Name=${player.controller}`, pawnRegExp);

  // get player position from pawn
  const posRegExp = new RegExp(`CapsuleComponent .+?PersistentLevel\\.${pawn}\\.CollisionCylinder\\.RelativeLocation = \\(X=(?<x>[\\d\\.-]+),Y=(?<y>[\\d\\.-]+),Z=(?<z>[\\d\\.-]+)\\)`);
  const { x, y, z } = await watch(`GetAll SceneComponent RelativeLocation Name=CollisionCylinder Outer=${pawn}`, posRegExp);

  return [x, y, z].map(Number);
}

// emit a console log
const log = (...args) => rpc.notify('log', args.join(' '));

// when available players updates - plugin:players:raw is emitted
rpc.addMethod('plugin:players:raw', ([playerArr]) => {
  // update the players list
  players = playerArr.map(p => ({
    name: p[0],
    id: p[1],
    controller: p[2],
    state: p[3],
  }));
});

// ping command
rpc.addMethod('chatcmd:ping', ([name, ...args]) => {
  rpc.notify('broadcast', `pong @ ${name} + ${args.length} args`);
});

// player position command
rpc.addMethod('chatcmd:pos', async ([name]) => {
  log ('player', name, 'requests position');
  const [x, y, z] = await getPlayerPos(name);
  rpc.notify('broadcast', `<b>${name}</> is at ${x} ${y} ${z}`);
});

// pass lines into the event emitter
rpc.addMethod('line', ([line]) => {
  ev.emit('line', line);
});

rpc.addMethod('init', async () => 'ok');
rpc.addMethod('stop', async () => 'ok');


```