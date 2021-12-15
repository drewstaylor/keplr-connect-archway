<template>
  <img alt="logo" src="./assets/logo.svg">
  
  <div class="content">

    <!-- Status Display / User Feedback -->
    <div class="status-display">
      <ul class="status wallet-status">
        <li class="counter"><strong>Wallet State:</strong>&nbsp;{{wallet.state}}</li>
      </ul>

    </div>

    <!-- Controls -->
    <div class="button-controls">
      <button id="connect" class="btn btn-main" @click="connectWallet();" v-if="wallet.state !== wallet.states[2]">Connect Wallet</button>
    </div>

    <!-- Loading -->
    <div class="loading" v-if="loading.status">
      <p v-if="loading.msg">{{loading.msg}}</p>
    </div>
  </div>
</template>

<script>
import { SigningCosmWasmClient } from '@cosmjs/cosmwasm-stargate';
import { chainInfo } from './chain.info'

const WALLET_STATES = ["Not connected", "Keplr not installed", "Connected", "Please update Keplr version"];

// var window;

export default {
  name: 'App',
  data: () => ({
    loading: {
      status: false,
      msg: ""
    },
    wallet: {
      state: WALLET_STATES[0],
      states: WALLET_STATES
    },
    offlineSigner: null,
    wasmClient: null,
    accounts: null
  }),
  mounted: async function () {
    // Init dApp
    await this.init();
  },
  methods: {
    init: async function () {
      console.log('Init');
    },
    connectWallet: async function () {
      console.log('Connecting wallet...', window);
      try {
        if (window) {
          if (window['keplr']) {
            console.log('window.keplr', window.keplr);
            if (window.keplr['experimentalSuggestChain']) {
              let chainId = chainInfo.chainId;
              let rpc = chainInfo.rpc;
              await window.keplr.experimentalSuggestChain(chainInfo)
              await window.keplr.enable(chainId);
              this.offlineSigner = await window.getOfflineSigner(chainId);
              this.wasmClient = await SigningCosmWasmClient.connectWithSigner(rpc, this.offlineSigner);
              this.accounts = await this.offlineSigner.getAccounts();

              if (this.accounts.length) {
                this.wallet.state = WALLET_STATES[2];
              }

              console.log('Wallet connected', {
                offlineSigner: this.offlineSigner,
                wasmClient: this.wasmClient,
                accounts: this.accounts
              });
            } else {
              console.warn('Error access experimental features, please update Keplr');
              this.wallet.state = WALLET_STATES[3];
            }
          } else {
            console.warn('Error accessing Keplr');
            this.wallet.state = WALLET_STATES[1];
          }
        } else {
          console.warn('Error parsing window object');
        }
      } catch (e) {
        console.error('Error connecting to wallet', e);
        this.wallet.state = WALLET_STATES[1];
      }
    }
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
div.content {
  text-align: left;
  margin: auto;
  margin-top: 3em;
  width: 90vw;
  max-width: 1280px;
}
div.content ul {
  list-style: none;
  padding-left: 0;
}
button {
  margin: 1rem;
  padding: 0.25rem;
}
p.label {
  padding-bottom: 0;
  margin-bottom: 0;
}
pre {
  line-height:1.2em;
  background:linear-gradient(180deg,#ccc 0,#ccc 1.2em,#eee 0);
  background-size:2.4em 2.4em;
  background-origin:content-box;
  padding: 1em;
  text-align:justify;
  display: inline-block;
  color: #0a4862;
  background-color: #73c8eb;
  border-color: #3bb3e3;
  border-radius: 0.5em;
}
</style>
