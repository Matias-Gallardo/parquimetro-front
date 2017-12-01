<template>
  <div class="hello">
    <b-container fluid class="bv-example-row">
      <b-row>
        <b-col cols="12">
          <h4>{{ msg }}</h4>
          <br>
        </b-col>
      </b-row>
      <b-row>
        <b-col cols="4" class="leftText">
          <table class="table table-bordered">
            <tr>
              <td>Contract address: {{ myContractAddress}}</td>
            </tr>
            <tr>
              <td>Contract owner: {{ owner}}</td>
            </tr>
            <tr>
              <td>Minimum Escrow: {{ cost }}</td>
            </tr>
            <tr>
              <td>Balance user: {{ balance }} -- Coins: {{ coins }}</td>
            </tr>
            <tr>
              <td>CityBalance: {{ cityBalance }}</td>
            </tr>
          </table>
        </b-col>
        <b-col cols="8">
          <b-form-group class="leftText">
            <span>Estado: {{(parked)?"Estacionado":"No estacionado" }}</span>
            <b-button v-on:click="isParked()" class="">is User Parked?</b-button>
          </b-form-group>

          <b-input-group>
            <b-input  class="col-md-4" type="text" v-model="money"></b-input>
            <b-input class="col-md-4" type="text" v-model="location"></b-input> &nbsp;
            <b-button v-on:click="startParking()">Start Parking</b-button>
          </b-input-group>
          <br>
          <b-form-group class="leftText">
            <b-button v-on:click="stopParking()">Stop Parking</b-button>
          </b-form-group>
        </b-col>
      </b-row>
    </b-container>


    <div id="transactions">
      <h4>Transactions</h4>
      <table class="table table-bordered">
        <thead>
        <th>From</th>
        <th>Tx Hash</th>
        <th>Tx Block Number</th>
        <th>Tx Gas Used</th>
        </thead>
        <tr v-for="(item, index) in transactions">
          <td>{{defaultSender}}</td>
          <td>{{ item.transactionHash }}</td>
          <td>{{ item.blockNumber }}</td>
          <td>{{ item.gasUsed }}</td>
        </tr>
      </table>
      <h4>Events</h4>
      <table class="table table-bordered">
        <thead>
        <th>Event</th>
        <th>Data</th>
        </thead>
        <tr v-for="(item, index) in transactions">
          <td>{{ getEventName(item.events)}}</td>
          <td>{{ getEventData(item.events)}}</td>
        </tr>
      </table>
    </div>
    <h4>{{ stopParkingInfo }}</h4>
  </div>
</template>

<script>
  import Web3 from 'web3';

  export default {
    name: 'HelloWorld',
    data() {
      return {
        msg: 'Welcome to your DappMeter',
        myContractAddress: '0xf25186b5081ff5ce73482ad761db0eb0d25abfbf',
        location: 'arbol.casa.bitcoin',
        balance: 0,
        web3: null,
        owner: null,
        contract: null,
        abi: null,
        defaultSender: null,
        money: 5760000,
        startParkingInfo: '',
        stopParkingInfo: '',
        hourPrice: 0,
        maxHours: 0,
        cost: 0,
        coins: 0,
        cityWallet: '0x5aeda56215b167893e80b4fe645ba6d5bab767de',
        parked: false,
        cityBalance: 0,
        transactions: [],
        events: [],
        eventManager: null,
      }
    },
    async created() {
      this.web3 = await new Web3(new Web3.providers.HttpProvider("http://localhost:9545"));
      this.connectToContract();
    },
    methods: {
      getEventName(item) {
        return Object.keys(item)[0];
      },
      getEventData(item) {
        if (this.getEventName(item) == "LogCarHasParked") {
          return "USER: " + item.LogCarHasParked.returnValues.user + " STARTIME: " + item.LogCarHasParked.returnValues.startTime + " LOCATION: " + item.LogCarHasParked.returnValues.location;
        } else {
          return "USER: " + item.LogCarHasLeft.returnValues.user + " COST: " + item.LogCarHasLeft.returnValues.cost + " TIME: " + item.LogCarHasLeft.returnValues.time;
        }
      },
      async startParking() {
        try {
          this.transactions.push(await this.contract.methods.startParking(this.location).send({
            from: this.defaultSender,
            value: this.money,
            gas: this.cost
          }));
          this.balance = await this.web3.eth.getBalance(this.defaultSender);
        } catch (exception) {
          alert("El auto ya se encuentra estacionado");
        }
      },
      async stopParking() {
        try {
        this.transactions.push(await this.contract.methods.stopParking().send({
          from: this.defaultSender,
          gas: this.cost
        }));
        this.balance = await this.web3.eth.getBalance(this.defaultSender);
        this.updateCoins();
        this.updateCityBalance();
        } catch(exception) {
           alert("El contador del estacionamiento no fue iniciado");
        }
      },
      async isParked() {
        this.parked = await this.contract.methods.isParked(this.defaultSender).call({from: this.defaultSender});
      },
      async connectToContract() {
        this.abi = await require('../assets/ParkingMeter.json');
        this.accounts = await this.web3.eth.getAccounts();
        this.defaultSender = this.accounts[0];
        this.web3.eth.defaultAccount = this.defaultSender;
        this.contract = await new this.web3.eth.Contract(this.abi.abi, this.myContractAddress);
        this.contract.defaultAccount = this.accounts[0];
        this.owner = await this.contract.methods.owner().call({from: this.defaultSender});
        this.balance = await this.web3.eth.getBalance(this.defaultSender);
        this.hourPrice = await this.contract.methods.getHourPrice().call({from: this.defaultSender});
        this.maxHours = await this.contract.methods.getMaxHours().call({from: this.defaultSender});
        this.cost = this.hourPrice * this.maxHours;
        this.updateCoins();
        this.cityWallet = await this.contract.methods.setCityWallet(this.accounts[1]).send({from: this.defaultSender});
        this.cityWallet = await this.contract.methods.cityWallet().call({from: this.defaultSender});
        this.updateCityBalance();
        //this.eventManager = await new Web3(new Web3.providers.WebsocketProvider('ws://localhost:8546'));
      },
      async updateEvents(event) {
        this.events.push(event);
      },
      async updateCoins() {
        this.coins = await this.contract.methods.getCoins().call({from: this.defaultSender});
      },
      async updateCityBalance() {
        this.cityBalance = await this.contract.methods.cityWalletBalance().call({from: this.defaultSender});
      }
    }

  }
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
  h1, h2 {
    font-weight: normal;
  }

  ul {
    list-style-type: none;
    padding: 0;
  }

  li {
    display: inline-block;
    margin: 0 10px;
  }

  a {
    color: #42b983;
  }

  .leftText {
    text-align: left;
  }
</style>
