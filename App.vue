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
