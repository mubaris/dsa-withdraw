<template>
  <div class="center examplex">
    <vs-navbar center-collapsed>
      <template #right>
        <vs-navbar-item v-if="notConnected">
          <vs-button @click="connectWallet">Connect Wallet</vs-button>
        </vs-navbar-item>
        <vs-navbar-item v-else>
          <p>{{`DSA #${dsaId}`}}</p>
          <vs-button danger @click="reset">Disconnect</vs-button>
        </vs-navbar-item>
      </template>
    </vs-navbar>
    <div class="square">
      <div v-if="!notConnected">
        <div v-if="dsaId > 0">
          <vs-row class="mh" v-if="tokenDataFetched" align="center" direction="column">
            <div style="margin: 4% 0">
              <vs-row>Name -&nbsp;<span style="font-weight: bold">{{tokenData.name}}</span></vs-row>
              <vs-row>Symbol -&nbsp;<span style="font-weight: bold">{{tokenData.symbol}}</span></vs-row>
              <!-- <vs-row>Decimals - <span style="font-weight: bold">{{tokenData.decimals}}</span></vs-row> -->
              <vs-row>Balance -&nbsp;<span style="font-weight: bold">{{tokenData.humanBal}}</span></vs-row>
            </div>
            <div>
              <vs-row justify="center">
                <vs-input
                  v-model="tokenAmount"
                  :disabled="withdrawMax"
                  placeholder="Enter Amount of tokens to withdraw"
                  style="margin: 2% 2%"
                />
                <vs-input v-model="withdrawAddr" placeholder="Enter Address to Withdraw" style="margin: 2% 2%" />
              </vs-row>
              <vs-row justify="center" >
                <vs-switch success v-model="withdrawMax">Withdraw Max</vs-switch>
              </vs-row>
            </div>
            <div>
              <vs-row justify="center" direction="row" style="margin: 4% 0">
                <vs-button @click="withdraw">Withdraw</vs-button>
                <vs-button warn @click="resetToken">Reset Token</vs-button>
              </vs-row>
            </div>
          </vs-row>
          <div v-else>
            <vs-input v-model="tokenAddress" placeholder="Enter Token Address" style="margin: 4% 0" />
            <vs-row justify="center">
              <vs-button @click="loadToken">Load Token</vs-button>
            </vs-row>
          </div>
        </div>
        <div v-else>
          Loading...
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { defineComponent, ref } from '@nuxtjs/composition-api'
import Web3 from 'web3'
import Web3Modal from 'web3modal'
import WalletConnectProvider from '@walletconnect/web3-provider'
import DSA from 'dsa-sdk'
import BigNumber from 'bignumber.js'

import erc20 from './abi/erc20.js'

export default defineComponent({
  setup() {
    const providerOptions = {
      walletconnect: {
        package: WalletConnectProvider,
        options: {
          infuraId: '509d29fe62db4e41bcae541d7434637c'
        }
      }
    }

    const web3Modal = new Web3Modal({ providerOptions })

    const notConnected = ref(true)
    const dsaId = ref(0)
    const dsaAddr = ref('0x0000000000000000000000000000000000000000')
    const _web3 = ref(null)
    const _dsa = ref(null)
    const tokenAddress = ref('')
    const tokenData = ref({})
    const tokenDataFetched = ref(false)
    const tokenAmount = ref('')
    const withdrawAddr = ref('')
    const withdrawMax = ref(false)

    async function connectWallet() {
      let loading;
      try {
        const provider = await web3Modal.connect()

        const web3 = new Web3(provider)

        _web3.value = web3

        const dsa = new DSA(web3)

        const accounts = await web3.eth.getAccounts()

        const address = accounts[0]

        if (address.length > 0) {
          notConnected.value = false
        }

        loading = this.$vs.loading()

        const dsaAccounts = await dsa.getAccounts(address)

        if (dsaAccounts.length > 0) {
          dsaId.value = dsaAccounts[0].id
          dsaAddr.value = dsaAccounts[0].address
          dsa.setInstance(dsaId.value)

          _dsa.value = dsa
        } else {
          this.$vs.notification({
            title: 'No DSA Account',
            text: `You don't have a DSA account created. Please create one from defi.instadapp.io`,
            color: 'warning'
          })
        }
      } catch (error) {
        console.log(error)
        loading.close()
        return
      }
      loading.close()
    }

    async function reset() {
      dsaId.value = 0
      notConnected.value = true
      await _web3.value.currentProvider.sendAsync({
        method: 'wc_killSession',
        params: []
      })
    }

    function getInt(v, d) {
      let valUnit = new BigNumber(v).multipliedBy(10 ** d)
      valUnit = valUnit.toFixed(0)
      return valUnit
    }

    async function withdraw() {
      const loading = this.$vs.loading()
      const dsa = _dsa.value
      const web3 = _web3.value
      const amount = Number.parseFloat(tokenAmount.value)
      if ((!amount || amount > tokenData.value.humanBal) && !withdrawMax.value) {
        this.$vs.notification({
          title: 'Error',
          text: 'Enter a valid number below the balance',
          color: 'danger'
        })
        loading.close()
        return
      }
      if (!web3.utils.isAddress(withdrawAddr.value.toLowerCase())) {
        this.$vs.notification({
          title: 'Error',
          text: 'Enter a valid address',
          color: 'danger'
        })
        loading.close()
        return
      }
      const addr = web3.utils.toChecksumAddress(withdrawAddr.value)
      let tokenAmt
      if (withdrawMax.value) {
        tokenAmt = dsa.maxValue
      } else {
        tokenAmt = getInt(amount, tokenData.value.decimals)
      }
      const spells = dsa.Spell()
      spells.add({
        connector: "basic",
        method: "withdraw",
        args: [tokenAddress.value, tokenAmt, addr, 0, 0]
      })
      try {
        await dsa.cast(spells)
      } catch (error) {
        console.log(error)
      }
      loading.close()
    }

    async function resetToken() {
      tokenAddress.value = ''
      tokenData.value = {}
      tokenDataFetched.value = false
      tokenAmount.value = ''
      withdrawAddr.value = ''
    }

    async function getToken(addr) {
      const web3 = _web3.value
      const contract = new web3.eth.Contract(erc20, addr)
      try {
        const decimals = await contract.methods.decimals().call()
        const symbol = await contract.methods.symbol().call()
        const name = await contract.methods.name().call()
        const balance = await contract.methods.balanceOf(dsaAddr.value).call()
        const humanBal = balance / (10 ** decimals)
        tokenData.value = { decimals, symbol, name, balance, humanBal }
        tokenDataFetched.value = true
      } catch (error) {
        this.$vs.notification({
          title: 'Error',
          text: 'Error fetching token data',
          color: 'danger'
        })
      }
    }

    async function loadToken() {
      const loading = this.$vs.loading()
      const web3 = _web3.value
      if (web3.utils.isAddress(tokenAddress.value.toLowerCase())) {
        const addr = web3.utils.toChecksumAddress(tokenAddress.value)
        await getToken(addr)
      } else {
        this.$vs.notification({
          title: 'Invalid Address',
          text: 'Please enter a valid address',
          color: 'warning'
        })
      }
      loading.close()
    }

    return {
      connectWallet,
      notConnected,
      dsaId,
      reset,
      tokenAddress,
      loadToken,
      tokenDataFetched,
      tokenData,
      tokenAmount,
      withdrawAddr,
      withdraw,
      resetToken,
      withdrawMax
    }
  }
})
</script>

