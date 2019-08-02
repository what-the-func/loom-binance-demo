<template>
  <v-layout column>
    <v-flex xs12 sm8 md6>
      <h1>BNB Deposit Address: tbnb1gc7azhlup5a34t8us84x6d0fluw57deuf47q9w</h1>
    </v-flex>
    <v-flex xs12 sm8 md6>
      <h1>Loom Address: {{ loomUserAddress }}</h1>
    </v-flex>
    <v-flex xs12 sm8 md6>
      <h1>Loom BNB Balance: {{ bnbBalance }}</h1>
    </v-flex>
    <v-flex xs12 sm6 md6 mt-5>
      <v-text-field
        v-model="binanceAddress"
        placeholder="Binance Address"
        solo
      />
    </v-flex>
    <v-flex xs12 sm6 md6>
      <v-btn @click="withdrawBNB">
        Withdraw BNB
      </v-btn>
    </v-flex>
  </v-layout>
</template>

<script>
import {
  Client,
  LocalAddress,
  CryptoUtils,
  LoomProvider,
  Address,
  createDefaultTxMiddleware
} from 'loom-js'
import Web3 from 'web3'
import { BinanceTransferGateway } from 'loom-js/dist/contracts'
import bech32 from 'bech32'
import BN from 'bn.js'
import bnbToken from '~/contracts/BNBToken.json'

export default {
  data() {
    return {
      client: null,
      binanceAddress: '',
      bnbBalance: 0,
      loomBNBContract: null,
      loomUserAddress: null,
      loomBNBGateway: null,
      privateKey: null,
      publicKey: null,
      web3: null
    }
  },

  async mounted() {
    this.createClient()
    this.getLoomUserAddress()
    this.getWeb3Instance()
    this.getLoomBNBContract()
    await this.getLoomBNBTransferGatewayContract()
    await this.filterEvents()
    await this.refreshBalance()
  },

  methods: {
    createClient() {
      this.privateKey = this.getPrivateKey()
      this.publicKey = CryptoUtils.publicKeyFromPrivateKey(this.privateKey)
      const writeUrl = 'wss://extdev-plasma-us1.dappchains.com/websocket'
      const readUrl = 'wss://extdev-plasma-us1.dappchains.com/queryws'
      const networkId = 'extdev-plasma-us1'
      this.client = new Client(networkId, writeUrl, readUrl)
      this.client.on('error', (msg) => {
        console.error('Error connecting to extdev.', msg)
      })
      this.client.txMiddleware = createDefaultTxMiddleware(
        this.client,
        this.privateKey
      )
    },

    getPrivateKey() {
      let privateKey = localStorage.getItem('loom_bnb_pk')
      if (!privateKey) {
        privateKey = CryptoUtils.generatePrivateKey()
        localStorage.setItem(
          'loom_bnb_pk',
          JSON.stringify(Array.from(privateKey))
        )
      } else {
        privateKey = new Uint8Array(JSON.parse(privateKey))
      }
      return privateKey
    },

    getLoomUserAddress() {
      this.loomUserAddress = LocalAddress.fromPublicKey(
        this.publicKey
      ).toString()
    },

    getWeb3Instance() {
      this.web3 = new Web3(new LoomProvider(this.client, this.privateKey))
    },

    getLoomBNBContract() {
      const bnbCoinAddress = '0x9ab4e22d56c0c4f7d494442714c82a605d2f28e0'
      this.loomBNBContract = new this.web3.eth.Contract(
        bnbToken.abi,
        bnbCoinAddress
      )
    },

    async filterEvents() {
      await this.loomBNBContract.events.Transfer(
        { filter: { address: this.loomUserAddress } },
        async (err, event) => {
          if (err) console.error('Error on event', err)
          await this.refreshBalance()
        }
      )
    },

    async refreshBalance() {
      this.bnbBalance = await this.loomBNBContract.methods
        .balanceOf(this.loomUserAddress)
        .call({ from: this.loomUserAddress })
      this.bnbBalance = this.bnbBalance / 100000000
    },

    async getLoomBNBTransferGatewayContract() {
      this.loomBNBGateway = await BinanceTransferGateway.createAsync(
        this.client,
        Address.fromString('extdev-plasma-us1:' + this.loomUserAddress)
      )
    },

    async withdrawBNB() {
      if (!this.binanceAddress) {
        console.error('Must provide a withdrawal address')
        return
      }

      const amountInt = parseFloat(this.bnbBalance) * 100000000
      const fee = 37500 // Binance Fee
      const loomBNBTransferGatewayAddress =
        '0xf801deec09eddf70b81e054c2241ece5f49edac2'
      await this.loomBNBContract.methods
        .approve(loomBNBTransferGatewayAddress, amountInt)
        .send({ from: this.loomUserAddress })
      const delay = (ms) => {
        // eslint-disable-next-line no-new
        new Promise((resolve) => {
          setTimeout(resolve, ms)
        })
      }
      let approvedBalance = 0
      while (approvedBalance === 0) {
        approvedBalance = await this.loomBNBContract.methods
          .allowance(this.loomUserAddress, loomBNBTransferGatewayAddress)
          .call({ from: this.loomUserAddress })
        await delay(5000)
      }
      const bnbTokenAddress = Address.fromString(
        'extdev-plasma-us1:' + this.loomBNBContract._address.toLowerCase()
      )
      const tmp = this.decodeAddress(this.binanceAddress)
      const recipient = new Address('binance', new LocalAddress(tmp))
      await this.loomBNBGateway.withdrawTokenAsync(
        new BN(amountInt - fee, 10),
        bnbTokenAddress,
        recipient
      )
    },

    decodeAddress(value) {
      const decodeAddress = bech32.decode(value)
      return Buffer.from(bech32.fromWords(decodeAddress.words))
    }
  }
}
</script>
