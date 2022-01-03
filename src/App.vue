<template>
  <img alt="logo" src="./assets/logo.svg">
  
  <div class="content">

    <!-- Status Display / User Feedback -->
    <div class="status-display">
      <ul class="status wallet-status">
        <li class="wallet-state">
          <strong>Wallet State:</strong>&nbsp;
          <span>{{wallet.state}}</span>
        </li>
      </ul>

      <!-- Account Info -->
      <div class="status-display accounts" v-if="accounts && wallet.state == wallet.states[2]">
        <div v-if="accounts.length">
          <ul class="status accounts-list">
            <li class="accounts account-item" v-for="(account, i) in accounts" :key="i">
              <!-- Address -->
              <strong>Account:</strong>&nbsp;
              <span>{{account.address}}</span>
              <!-- Balance -->
              <div v-if="account.balance">
                <strong v-if="!isNaN(account.balance.amount)">Balance:</strong>&nbsp;
                <span v-if="!isNaN(account.balance.amount)">{{parseInt(account.balance.amount) / 100000}} {{chainMeta.currencies[0].coinDenom}}</span>
              </div>
            </li>
          </ul>
        </div>
      </div>
    </div>

    <!-- Controls -->
    <div class="button-controls">
      <!-- Connect -->
      <button id="connect" class="btn btn-main" @click="connectWallet();" v-if="wallet.state !== wallet.states[2]">Connect Wallet</button>
      <!-- Request -->
      <button id="request" class="btn btn-main" @click="requestFaucet();" v-if="wallet.state == wallet.states[2] && !loading && status.type !== 'success'">Request Faucet Funds</button>
    </div>

    <!-- Loading -->
    <div class="loading bg-info" v-if="loading">
      <p v-if="loading">{{loading}}</p>
    </div>

    <!-- Success / Error -->
    <div v-if="status.type && status.msg">
      <div class="close-x">
        <span class="float-right" @click="status = {type:null,msg:null}">×</span>
      </div>
      <div class="bg-success" v-if="status.type == 'success'">{{status.msg}}</div>
      <div class="bg-danger" v-if="status.type == 'danger'">{{status.msg}}</div>
    </div>

  </div>
</template>

<script>
import { SigningCosmWasmClient } from '@cosmjs/cosmwasm-stargate';
import { chainInfo } from './chain.info';
import axios from 'axios';

const WALLET_STATES = ["Not connected", "Keplr not installed", "Connected", "Please update Keplr version"];

export default {
  name: 'App',
  data: () => ({
    wallet: {
      state: WALLET_STATES[0],
      states: WALLET_STATES
    },
    offlineSigner: null,
    wasmClient: null,
    accounts: null,
    chainMeta: chainInfo,
    loading: null,
    status: {
      type: null,
      msg: null
    }
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
      console.log('Connecting wallet...');
      try {
        if (window) {
          if (window['keplr']) {
            if (window.keplr['experimentalSuggestChain']) {
              let chainId = chainInfo.chainId;
              let rpc = chainInfo.rpc;
              await window.keplr.experimentalSuggestChain(chainInfo)
              await window.keplr.enable(chainId);
              this.offlineSigner = await window.getOfflineSigner(chainId);
              this.wasmClient = await SigningCosmWasmClient.connectWithSigner(rpc, this.offlineSigner);
              this.accounts = await this.offlineSigner.getAccounts();

              console.log('Wallet connected', {
                offlineSigner: this.offlineSigner,
                wasmClient: this.wasmClient,
                accounts: this.accounts,
                chain: this.chainMeta
              });

              if (this.accounts.length) {
                this.wallet.state = WALLET_STATES[2];
                await this.getBalances();
              }
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
    },
    getBalances: async function () {
      if (!this.chainMeta) {
        return;
      } else if (!this.chainMeta['chainName']) {
        return;
      }

      this.loading = 'Updating account balances...';
      this.status = {
        type: null,
        msg: null
      };

      if (this.accounts) {
        if (this.accounts.length) {
          for (let i = 0; i < this.accounts.length; i++) {
            if (this.accounts[i]) {
              if (this.accounts[i]['address']) {
                try {
                  this.accounts[i].balance = await this.wasmClient.getBalance(this.accounts[i].address, 'uconst');
                  console.log('Balance', this.accounts[i].balance);
                  this.loading = null;
                } catch (e) {
                  console.warn('Error reading account address', [e, this.accounts[i]]);
                  this.loading = null;
                  this.status = {
                    type: 'error',
                    msg: String(e)
                  };
                  return;
                }
              }
            }
          }
        } else {
          this.loading = null;
          this.status = {
            type: 'warning',
            msg: 'Failed to resolve Keplr wallet'
          };
        }
      } else {
        this.loading = null;
        this.status = {
          type: 'warning',
          msg: 'Failed to resolve Keplr wallet'
        };
      }
    },
    requestFaucet: async function () {
      if (!this.chainMeta) {
        return;
      } else if (!this.chainMeta['chainName']) {
        return;
      } else if (!this.chainMeta['currencies']) {
        return;
      } else if (!this.chainMeta.currencies.length) {
        return;
      } else if (!this.chainMeta['faucets']) {
        return;
      } else if (!this.chainMeta.faucets.length) {
        return;
      }

      this.loading = 'Requesting faucet funds from ' + String(this.chainMeta.chainName);
      this.status = {
        type: null,
        msg: null
      };

      const apiClient = axios.create();

      try {
        if (this.accounts) {
          if (this.accounts.length) {
            for (let i = 0; i < this.accounts.length; i++) {
              let request = {
                "denom": this.chainMeta.currencies[0].coinMinimalDenom,
                "address": this.accounts[i].address
              };

              let res = await apiClient.post(this.chainMeta.faucets[0], request);             
              console.log('Faucet request result', res);
              
              this.loading = null;
              this.status = {
                type: 'success',
                msg: 'Your request for faucet funds has been submitted'
              };
            }
          } else {
            this.loading = null;
            this.status = {
              type: 'warning',
              msg: 'Failed to resolve Keplr wallet'
            };
          }
        } else {
          this.loading = null;
          this.status = {
            type: 'warning',
            msg: 'Failed to resolve Keplr wallet'
          };
        }
      } catch (e) {
        this.loading = null;
        this.status = {
          type: 'error',
          msg: String(e)
        };
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
  padding: 0.25rem;
}
li span {
  font-style: italic;
}
.loading.bg-info,
.bg-success,
.bg-danger {
  margin-top: 2rem;
  color: #ffffff;
}
</style>
