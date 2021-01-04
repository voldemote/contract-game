# 2021/01/01

## Our prototype game contract


## Implementing our contract

ink_storage::HashMap doesn't implement Clone.

We try to put an `ink_storage::HashMap` into a custom `GameAccount` struct
inside another `ink_storage::HashMap`, inside our `Game` storage type.

Something like

```rust
    #[ink(storage)]
    pub struct Game {
        game_accounts: ink_storage::HashMap<AccountId, GameAccount>,
    }

    #[derive(Debug, Clone, scale::Encode, scale::Decode, ink_storage_derive::PackedLayout, ink_storage_derive::SpreadLayout)]
    pub struct GameAccount {
        level: u32,
        level_programs: ink_storage::HashMap<u32, AccountId>,
    }
```

It doesn't work, with a bunch of errors about traits like `WrapperTypeEncode`
not being implemented for types like `ink_storage::HashMap`.

We look at the ink examples and don't see any using nested collection
types in their storage type.
Instead they all use a "flat" data model.
I don't really want to do that because it will be harder to maintain
the invariants I want.
Reading the API docs for `scale::Encode` we see that the standard
`BTreeMap` type implements it,
so instead of nesting `ink_storage::HashMap`s inside each other,
we use a `BTreeMap` instead, like

```rust
    #[ink(storage)]
    pub struct Game {
        game_accounts: ink_storage::HashMap<AccountId, GameAccount>,
    }

    #[derive(Debug, Clone, scale::Encode, scale::Decode, ink_storage_derive::PackedLayout, ink_storage_derive::SpreadLayout)]
    pub struct GameAccount {
        level: u32,
        level_programs: BTreeMap<u32, AccountId>,
    }
```

I imagine this will be inefficient,
but for now we want our code to be readable,
not efficient.


For each level of our game,
our game contract needs to call a player's "level contract".
So each of our levels defines a contract interface,
and each player implements that interface in their own
contract for the level.

When we start trying to figure out how to call another
arbitrary contract,
using only some interface,
we run into a lack of examples and documentation.




While incrementally adding features,
experimenting with ink APIs,
and attempting to debug,
we find that we don't know how to do "println debugging":
ink defines [`ink_env::debug_println`],
but when we use it we don't see any output anywhere.

[`ink_env::debug_println`]: https://paritytech.github.io/ink/ink_env/fn.debug_println.html

I aske in the `#ink:matrix.parity.io` channel where to see the output,
and Alexander Theißen replies:

> "They are printed to the console. You need to raise the log level for `runtime`
  module in order to see them. I also recommend to silence all the block
  production noise: `-lerror,runtime=debug`"

So those are presumably flags to `canvas-node`. TODO




## Connecting to our contract with polkadot-js

It's strange that the JS compononts are "polkadot"-branded,
where so many other things in this ecosystem are "substrate"-branded.

I try not to use npm unless I have to.
Trying to create a simple frontend using plain HTML and JavaScript.
All the documentation for the polkadot JS API assumes the use of node/yarn.
I am trying to figure out how to use webpack to package up @polkadot/api so I can use it outside npm,
but don't know how.

I have previously succeeded in this with web3.js and ipfs.js,
but don't really remember how,
and don't see any obvious evidence that the polkadot APIs are ready to webpack.

I ask in #ink:matrix.parity.io

> I'm not very familiar with npm programming, and I want to use the
  @polkadot/api from a non-npm script in the browser. Is it possible to use
  webpack to package @polkadot/api into a standalone javascript file that I can
  use outside of npm. Any hints how to do that?

In the meantime I give up trying to package polkadot-js for use
outside of npm,
and try to set up a yarn app that will let me import the the library
in the expected way.

As someone mostly unfamiliar with npm I immediately encounter more problems.
I know how to add `@polkadot/api` to a yarn project,
but I don't know how to set yarn up to run a server on `yarn start`.
From Googling, as with most things in the npm ecosystem,
there seem to be many options.

Similar to the Ethereum docs,
I'm finding that the polkadot-js docs completely gloss over topics
about setting up npm/parn projects,
and I am completely lost.

I know that I can't expect Polkadot to teach me npm,
just like I can't expect them to teach me Rust,
but this has been a huge problem for me every time I try to use modern JavaScript.

On https://polkadot.js.org/docs/api/examples/promise/ it says

> "From each folder, run yarn to install the required dependencies and then run
  yarn start to execute the example against the running node."

But there are no "folders" in this documentation.
Is there a link to actual, complete, ready-to-run source code I'm missing?

I'm quite frustrated.

I additionally ask in the "#ink" channel if there's a basic
yarn template for using the polkadot JS API's.

Dan Shields comes through with a link to

> https://github.com/substrate-developer-hub/substrate-front-end-template

I recall seeing this before and must have forgotten about it.
I'll try to reboot my web efforts from this template.

Well, maybe I'll just use it for hints.
It's a react app, and I really don't want to learn react right now.
So complex.

I am going to try instead using `webpack-dev-server` for my `yarn start` command.

I eventually follow the [webpack "Getting Started" guide][wpgs].
I'm real rak shaving now.

[wpgs]: https://webpack.js.org/guides/getting-started/

While running weback with my script that imports "@polkadot/api" I run into this error:

```
ERROR in ./node_modules/scryptsy/lib/utils.js 1:15-32
Module not found: Error: Can't resolve 'crypto' in '/home/ubuntu/contract-game/www2/node_modules/scryptsy/lib'

BREAKING CHANGE: webpack < 5 used to include polyfills for node.js core modules by default.
This is no longer the case. Verify if you need this module and configure a polyfill for it.

If you want to include a polyfill, you need to:
        - add a fallback 'resolve.fallback: { "crypto": require.resolve("crypto-browserify") }'
        - install 'crypto-browserify'
If you don't want to include a polyfill, you can use an empty module like this:
        resolve.fallback: { "crypto": false }
 ```

To fix this it seems I have to add a `webpack.config.js` and configure it per the
[webpack "resolve" instructions][wri].
After creating `webpack.config.js` I can more-or-less copy-paste the suggestion
straight from the command line.
My new `webpack.config.js` looks like

```js
const path = require("path");

module.exports = {
    entry: './src/index.js',
    output: {
        filename: 'main.js',
        path: path.resolve(__dirname, 'dist'),
    },
    resolve: {
        fallback: {
            "crypto": require.resolve("crypto-browserify")
        }
    }
};
```

I also have to `yarn add crypto-browserify`.
Once I do I see more similar errors about polyfills,
and when I finally have webpack working I have
three polyfills in my `webpack.config.js`:

```js
        fallback: {
            "buffer": require.resolve("buffer"),
            "crypto": require.resolve("crypto-browserify"),
            "stream": require.resolve("stream-browserify")
        }
```

[wri]: https://webpack.js.org/configuration/resolve/

Hours and hours go by...

I'm using webpack 5,
which doesn't do a bunch of node polyfills when it compiles
for the browser.
I think that's a big part of my pain.

After tons of Googling and hacking I finally manage
to load the polkadot JS API in my mostly vanilla JS
web page.

I have to do a lot of hacks.

At the end my `webpack.config.js` looks like

```js
const path = require("path");
const webpack = require("webpack");

module.exports = {
    mode: "development",
    entry: './src/index.js',
    output: {
        filename: 'js/bundle.js',
        path: path.resolve(__dirname, 'dist'),
    },
    resolve: {
        fallback: {
            "buffer": require.resolve("buffer"),
            "crypto": require.resolve("crypto-browserify"),
            "stream": require.resolve("stream-browserify")
        }
    },
    plugins: [
        new webpack.ProvidePlugin({
            Buffer: ['buffer', 'Buffer'],
        }),
    ]
};
```

That `plugins` section is new and mysterious,
just copy pasted from [some commit to some other
software project][webhack].

[webhack]: https://github.com/duplotech/create-react-app/commit/d0be703d40cd4bc32cd91128ba407a138c608243#diff-8e25c4f6f592c1fcfc38f0d43d62cbd68399f44f494c1b60f0cf9ccd7344d697

I also have this lovely garbage in my HTML header
before loading my webpack bundle:

```html
  <script>
    let global = window;
    let process = {
      "versions": null
    };
  </script>
```

Yup.

Somebody tell me what I'm doing wrong. Please.