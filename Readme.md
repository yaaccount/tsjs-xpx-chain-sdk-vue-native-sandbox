# MOTIVATION #

It is not a trivial procedure to setup a crypto javascript project working within vue/react native cross platform environment. This is just a PoC, which can serve as a base for other developers to avoid common roadblocks. You should be able to fork/checkout/clone/use directly this project. If something goes wrong, those are the steps I used to create it.

# PREREQUISITIES

I worked on my mac (tested also with ubuntu + real android device), I don't know what tools are essential for the procedure to work. Those are the obvious ones:

* Xcode / android-sdk
* Node + npm
  * My favourite setup is with ```nvm``` (node version manager) installed through ```brew``` - that way you can easily switch between different ```node``` versions - and install node using ```nvm```
  * I have v10.17.0 at the moment
* vue native - https://vue-native.io/docs/installation.html
* reactive native - https://reactnative.dev/docs/environment-setup (no expo)

# INSTALLATION from github using this repository

    $ git clone https://github.com/yaaccount/tsjs-xpx-chain-sdk-vue-native-sandbox.git
    $ cd tsjs-xpx-chain-sdk-vue-native-sandbox
    $ npm i
    $ react-native run-ios
    $ react-native run-android

# INSTALLATION step by step from scratch #

* In terminal:


      $ vue-native init TsjsXpxChainSdkVueNativeSandbox --no-expo
      $ cd TsjsXpxChainSdkVueNativeSandbox


* Following https://github.com/tradle/react-native-crypto

      $ npm install --save react-native-crypto
      $ npm install --save react-native-randombytes
      $ react-native link react-native-randombytes
      $ npm install --save-dev rn-nodeify
      $ npm install --save tsjs-xpx-chain-sdk asyncstorage-down

*  add this line into your package.json - scripts; alternatively, run the rn-nodeify command manually after npm install
  
      "postinstall": "./node_modules/.bin/rn-nodeify --install buffer,events,process,stream,util,inherits,fs,path,assert,crypto,net,url,vm,http,https,zlib,tls,os --hack",


* to trigger postinstall with rn-nodeify

      $ npm install

* rn-nodeify generated a shim.js file - add this line at the top of your index.js

      import './shim';

* add more code into your app, i.e. our tsjs-xpx-chain-sdk-vue-native-sandbox PoC in App.vue, you can copy/paste it directly; don't laugh - I suck at GUI really bad - this is all you can get from me

      <template>
        <view class="container">
          <text class="welcome">{{state.message}}</text>
          <button
            :on-press="sendDummyTx"
            title="Send hello world tx"
            color="red"
            accessibility-label="press to add one"
          />
          <text class="welcome">{{state.counter}}</text>
          <text class="transaction">tx status: {{state.tx.state ? state.tx.state : "unknown"}}</text>
          <text class="transaction">tx hash: {{state.tx.hash ? state.tx.hash : "unknown"}}</text>
          <text
            class="block"
          >last block received, height: {{state.lastBlock ? state.lastBlock.height.compact() : 0}}</text>
          <scroll-view :content-container-style="{contentContainer: {
              paddingVertical: 20
          }}">
            <text class="block">last block received, content: {{state.lastBlock}}</text>
          </scroll-view>
        </view>
      </template>

      <script>
      import {
        BlockHttp,
        Listener,
        Account,
        NetworkType,
        TransactionBuilderFactory,
        PlainMessage,
        TransactionHttp
      } from "tsjs-xpx-chain-sdk";
      // const APIUrl = "https://arcturus.xpxsirius.io";
      const APIUrl = "http://bctestnet1.brimstone.xpxsirius.io:3000";
      const blockHttp = new BlockHttp(APIUrl);
      const transactionHttp = new TransactionHttp(APIUrl);
      const listener = new Listener(APIUrl, global.WebSocket);

      const state = {
        initialized: false,
        server: APIUrl,
        message: "tsjs-xpx-chain-sdk demo",
        counter: 0,
        lastBlock: undefined,
        tx: {
          state: undefined,
          hash: undefined
        },
        factory: new TransactionBuilderFactory(),
        account: undefined
      };

      blockHttp.getBlockByHeight(1).subscribe(blockInfo => {
        console.log(blockInfo);
        state.lastBlock = blockInfo;

        state.factory = new TransactionBuilderFactory();
        state.factory.networkType = blockInfo.networkType;
        state.factory.generationHash = blockInfo.generationHash;

        state.account = Account.createFromPrivateKey(
          "a".repeat(64),
          state.factory.networkType
        );
        state.initialized = true;

        listener.open().then(() => {
          const newBlock = listener.newBlock().subscribe(bi => {
            console.log(bi);
            state.lastBlock = bi;
          });
        });
      });

      const sendTx = () => {
        if (!state.initialized) return;

        state.counter += 1;

        const tx = state.factory
          .transfer()
          .recipient(state.account.address)
          .mosaics([])
          .message(PlainMessage.create("hello"))
          .build();
        state.tx.state = "created";

        const signedTx = state.account.sign(tx, state.factory.generationHash);
        console.log(signedTx);
        state.tx.state = "signed";

        const status = listener.status(state.account.address).subscribe(error => {
          console.log(error);
          state.tx.state = error;
          status.unsubscribe();
          confirmed.unsubscribe();
        });

        const confirmed = listener
          .confirmed(state.account.address)
          .subscribe(confirmedTx => {
            if (
              confirmedTx.transactionInfo &&
              signedTx.hash === confirmedTx.transactionInfo.hash
            ) {
              console.log("our tx confirmed:");
              console.log(confirmedTx);
              state.tx.state = "confirmed";
              state.tx.hash = confirmedTx.transactionInfo.hash;
              status.unsubscribe();
              confirmed.unsubscribe();
            } else {
              console.log("other tx for us:");
              console.log(confirmedTx);
            }
          });

        transactionHttp.announce(signedTx).subscribe(
          response => {
            console.log(response);
            state.tx.state = "announced";
          },
          error => {
            state.tx.state = "failed announce";
            status.unsubscribe();
            confirmed.unsubscribe();
          },
          () => {}
        );
      };

      export default {
        data: () => {
          return {
            state
          };
        },
        methods: {
          sendDummyTx: sendTx
        }
      };
      </script>

      <style>
      .container {
        background-color: #333;
        align-items: center;
        justify-content: center;
        flex: 1;
      }
      .welcome {
        color: white;
        font-size: 30px;
        margin: 30px;
      }
      .message {
        color: greenyellow;
        font-size: 15px;
        margin: 15px;
      }
      .block {
        color: greenyellow;
        font-size: 15px;
        margin: 15px;
      }
      .transaction {
        color: greenyellow;
        font-size: 15px;
        margin: 15px;
      }
      </style>


# Additional hacks to make it work

1. flatbuffers

  * In

        node_modules/flatbuffers/js/flatbuffers.js

    replace

        this.flatbuffers = flatbuffers;

    with 

        export { flatbuffers };

2. tsjs-xpx-chain-sdk
  * file

        node_modules/tsjs-xpx-chain-sdk/dist/src/core/format/Utilities.js

    * checkout ```tsjs-xpx-chain-sdk``` from github
    * remove all ```this.``` occurences from

          src/core/format/Utilities.ts 

    * re-compile the library (```$ npm run build```)
    * copy resulting
          
          dist/src/core/format/Utilities.js
        
        here over
        
          node_modules/tsjs-xpx-chain-sdk/dist/src/core/format/Utilities.js


        (this will be probably fixed in the tsjs-xpx-chain-sdk in near future, for now you need to do it yourself)


          $ react-native run-ios
          $ react-native run-android


Happy coding!

