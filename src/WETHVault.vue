<template lang="pug">
  div(v-if="isDrizzleInitialized", id="app")
    h1 WETH Citadel
    div WETH price (CoinGecko 🦎): {{ weth_price | fromWei(4) | toCurrency(4) }}
    div Deposit Limit: {{ vault_deposit_limit | fromWei(2) }}
    div Total Assets: {{ vault_total_assets | fromWei(2) }}
    div Total AUM: {{ vault_total_aum | toCurrency(2) }}  
    p
    div Price Per Share: {{ vault_price_per_share | fromWei(8) }}
    div Available limit: {{ vault_available_limit | fromWei(2) }} WETH
    p
    div Your Account: <strong>{{ username || activeAccount }}</strong>
    div Your Vault shares: {{ yvweth_balance | fromWei(2) }}
    div Your WETH Balance: {{ weth_balance | fromWei(2) }}
    div Your ETH Balance: {{ eth_balance | fromWei(2) }}
    label(v-if="vault_available_limit > 0") Wrap ETH 
    input(v-if="vault_available_limit > 0" size="is-small" v-model.number="amount_wrap" type="number" min=0)
    | 
    button(:disabled="!eth_balance > 0" , @click.prevent='on_wrap_eth') 🍬 Wrap
    div.spacer
    div(v-if="total_yfi >= entrance_cost")
      span <strong>You are a guest. Welcome to the <span class="blue">Citadel</span> 🏰</strong>
      p
      label(v-if="vault_available_limit > 0") Amount 
      input(v-if="vault_available_limit > 0" size="is-small" v-model.number="amount" type="number" min=0)
      span(v-if="vault_available_limit <= 0") Deposits closed. 
      p
      button(v-if="vault_available_limit > 0" :disabled='has_allowance_vault', @click.prevent='on_approve_vault') {{ has_allowance_vault ? '✅ Approved' : '🚀 Approve Vault' }}
      button(v-if="vault_available_limit > 0" :disabled='!has_allowance_vault', @click.prevent='on_deposit') 🏦 Deposit
      button(v-if="vault_available_limit > 0" :disabled='!has_allowance_vault', @click.prevent='on_deposit_all') 🏦 Deposit All
      button(:disabled='!has_weth_balance', @click.prevent='on_withdraw_all') 💸 Withdraw All
    div(v-else)
      div.red
        span ⛔ You need {{ entrance_cost - total_yfi }} YFI more to enter the Citadel ⛔
      <div v-konami @konami="bribe_unlocked = !bribe_unlocked"></div>
      div(v-if="bribe_unlocked")
        span If you still want to join the party...
        | 
        button(@click.prevent='on_approve_vault') 💰 Bribe the bouncer
      div(v-else)
        span Remember Konami 🎮
    div.red(v-if="error")
      span {{ error }}
    p
      div.muted
        span Made with 💙  
        span yVault:
        | 
        a(href='https://twitter.com/arbingsam' target='_blank') arbingsam
        span  - UI:
        | 
        a(href='https://twitter.com/fameal', target='_blank') fameal
  div(v-else)
    div Loading yApp...

</template>

<script>
import { mapGetters } from 'vuex'
import ethers from 'ethers'
import axios from 'axios'
import GuestList from './abi/GuestList.json'
import yVaultV2 from './abi/yVaultV2.json'
import Web3 from 'web3'
let web3 = new Web3(Web3.givenProvider);

const max_uint = new ethers.BigNumber.from(2).pow(256).sub(1).toString()
//const BN_ZERO = new ethers.BigNumber.from(0)
const ERROR_NEGATIVE = "You have to deposit a positive number of tokens 🐀"
const ERROR_NEGATIVE_ALL = "You don't have tokens to deposit 🐀"
const ERROR_NEGATIVE_WITHDRAW = "You don't have any vault shares"
const ERROR_GUEST_LIMIT = "That would exceed your guest limit. Try less."
const ERROR_GUEST_LIMIT_ALL = "That would exceed your guest limit. Try not doing all in."


export default {
  name: 'WETHVault',
  data() {
    return {
      username: null,
      weth_price: 0,
      amount: 0,
      amount_wrap: 0,
      error: null,
      contractGuestList: null,
      is_guest: false,
      entrance_cost: 1,
      total_yfi: 0.5,
      bribe_unlocked: false
    }
  },
  filters: {
    fromWei(data, precision) {
      if (data === 'loading') return data
      if (data > 2**255) return '♾️'
      let value = ethers.utils.commify(ethers.utils.formatEther(data))
      let parts = value.split('.')

      if (precision === 0) return parts[0]

      return parts[0] + '.' + parts[1].slice(0, precision)
    },
    toPct(data, precision) {
      if (isNaN(data)) return '-'
      return `${(data * 100).toFixed(precision)}%`
    },
    toCurrency(data, precision) {
      if (typeof data !== "number") {
        data = parseFloat(data);
      }
      var formatter = new Intl.NumberFormat('en-US', {
          style: 'currency',
          currency: 'USD',
          minimumFractionDigits: precision
      })
      return formatter.format(data);
    },
  },
  methods: {
    on_approve_vault() {
      this.drizzleInstance.contracts['WETH'].methods['approve'].cacheSend(this.vault, max_uint, {from: this.activeAccount})
    },
    on_wrap_eth() {
      this.error = null

      if (this.amount_wrap <= 0) {
        this.error = ERROR_NEGATIVE
        this.amount_wrap = 0
        return
      }
    
    this.drizzleInstance.contracts['WETH'].methods['deposit'].cacheSend({from: this.activeAccount, value: ethers.utils.parseEther(this.amount_wrap.toString()).toString()})
    },
    on_deposit() {
      this.error = null

      if (this.amount <= 0) {
        this.error = ERROR_NEGATIVE
        this.amount = 0
        return
      }

      this.contractGuestList.methods.authorized(this.activeAccount, this.amount).call().then( response => {
        if (response === true) {
          this.drizzleInstance.contracts['WETHVault'].methods['deposit'].cacheSend(ethers.utils.parseEther(this.amount.toString()).toString(), {from: this.activeAccount})
        } else {
          this.error = ERROR_GUEST_LIMIT
        }
      })

    },
    on_deposit_all() {
      if (this.weth_balance <= 0) {
        this.error = ERROR_NEGATIVE_ALL
        this.amount = 0
        return
      }
      
      
      this.contractGuestList.methods.authorized(this.activeAccount, this.weth_balance).call().then( response => {
        if (response === true) {
          this.drizzleInstance.contracts['WETHVault'].methods['deposit'].cacheSend({from: this.activeAccount})
        } else {
          this.error = ERROR_GUEST_LIMIT_ALL
        }
      })

    },
    on_withdraw_all() {
      if (this.yvweth_balance <= 0) {
        this.error = ERROR_NEGATIVE_WITHDRAW
        this.amount = 0
        return
      }
      this.drizzleInstance.contracts['WETHVault'].methods['withdraw'].cacheSend({from: this.activeAccount})
    },
    async load_reverse_ens() {
      let lookup = this.activeAccount.toLowerCase().substr(2) + '.addr.reverse'
      let resolver = await this.drizzleInstance.web3.eth.ens.resolver(lookup)
      let namehash = ethers.utils.namehash(lookup)
      this.username = await resolver.methods.name(namehash).call()
    },
    call(contract, method, args, out='number') {
      let key = this.drizzleInstance.contracts[contract].methods[method].cacheCall(...args)
      let value
      try {
        value = this.contractInstances[contract][method][key].value 
      } catch (error) {
        value = null
      }
      switch (out) {
        case 'number':
          if (value === null) value = 0
          return new ethers.BigNumber.from(value)
        case 'address':
          return value
        default:
          return value
      }
    },
  },
  computed: {
    ...mapGetters('accounts', ['activeAccount', 'activeBalance']),
    ...mapGetters('drizzle', ['drizzleInstance', 'isDrizzleInitialized']),
    ...mapGetters('contracts', ['getContractData', 'contractInstances']),

    user() {
      return this.activeAccount
    },
    vault() {
      return this.drizzleInstance.contracts['WETHVault'].address
    },
    strategy_01() {
      return this.drizzleInstance.contracts['yDaiStrategy'].address
    },
    vault_supply() {
      return this.call('WETHVault', 'totalSupply', [])
    },
    vault_deposit_limit() {
      return this.call('WETHVault', 'depositLimit', [])
    },
    vault_total_assets() {
      return this.call('WETHVault', 'totalAssets', [])
    },
    vault_available_limit() {
      return this.call('WETHVault', 'availableDepositLimit', [])
    },
    vault_total_aum() {
      let toInt = new ethers.BigNumber.from(10).pow(18).pow(2).toString()
      return this.vault_total_assets.mul(this.weth_price).div(toInt)
    },
    vault_price_per_share() {
      return this.call('WETHVault', 'pricePerShare', [])
    },
    yvweth_balance() {
      return this.call('WETHVault', 'balanceOf', [this.activeAccount])
    },
    weth_balance() {
      return this.call('WETH', 'balanceOf', [this.activeAccount])
    },
    eth_balance(){
      return this.activeBalance
    },
    has_allowance_vault() {
      return !this.call('WETH', 'allowance', [this.activeAccount, this.vault]).isZero()
    },
    has_weth_balance() {
      return (this.weth_balance > 0)
    },
  },
  async created() {

    axios.get('https://api.coingecko.com/api/v3/simple/price?ids=weth&vs_currencies=usd')
      .then(response => {
        this.weth_price = ethers.utils.parseUnits(String(response.data.weth.usd),18)
      })

    //Active account is defined?
    if (this.activeAccount !== undefined) this.load_reverse_ens()
    
    // Get GuestList contract and use it :)
    let WETHVault = new web3.eth.Contract(yVaultV2, this.vault)
    WETHVault.methods.guestList().call().then( response => {
      this.contractGuestList = new web3.eth.Contract(GuestList, response)

      this.contractGuestList.methods.guests(this.activeAccount).call().then( response => {
        this.is_guest = response
      })

      /*this.contractGuestList.methods.total_yfi(this.activeAccount).call().then( response => {
        this.total_yfi = response
      })*/

      /*this.contractGuestList.methods.entrance_cost().call().then( response => {
        this.entrance_cost = response
      })*/

    })

  },

}
</script>

<style>
button {
  margin-right: 1em;
}
.muted {
  color: gray;
  font-size: 0.8em;
}
.red {
  color: red;
  font-weight: 700;
}
.blue {
  color: blue;
  font-weight: 700;
}
.spacer {
  padding-top: 1em;
  padding-bottom: 1em;
}
a, a:visited, a:hover {
  color: gray;
}
</style>