<template>
  <div class="hello">
    <h1>Plutus spend overview</h1>
    <p>
      Select your .csv statements file from Plutus (<a href="https://chrome.google.com/webstore/detail/plutusdex-enhancer/necjdfandaodcoeagkacmlapednbihgl" target="_blank">Get the plugin!</a>) to get a nice overview and possibilities of future spendings.<br>
      Don't have a Plutus account or just wondering wtf this is? <a href="https://dex.plutus.it/auth/referee/signup?refId=rx8znb" target="_blank" rel="noopener">Get one here (Ref link)</a>.<br>
      <a href="https://github.com/ensconce/plu.pay4.se">Check out the source here.</a>
    </p>
    
    <h4>Select your current tier</h4>

    <input type="radio" id="starter" value="starter" v-model="tier" @change="changeTier"/>
    <label for="starter">Starter</label>

    <input type="radio" id="everyday" value="everyday" v-model="tier" @change="changeTier"/>
    <label for="everyday">Everyday</label>

    <input type="radio" id="premium" value="premium" v-model="tier" @change="changeTier"/>
    <label for="premium">Premium</label>


    <h4>Select your statements file</h4>
    
    <label for="statementsFile">
      <div class="upload" style="border: 1px dashed #203653; margin-left: auto; margin-right: auto; cursor: pointer; padding-bottom: 25px;">
        <h4 style="color: #d1ae52; margin-bottom: 0px;">Select file</h4>
        <small style="font-size: 0.9em; margin-top: 0;">Nothing gets sent to any server, check the inspector!</small>
      </div>
    </label>
    <input id="statementsFile" type="file" accept=".csv" @change="handleFileUpload( $event )" style="display: none"/>

    <h3 style="color: #d1ae52">Spendable amount right now: 
    {{Math.min.apply(null,[
      (limit1day-spent1day).toFixed(2), 
      (limit7days-spent7days).toFixed(2), 
      (limit30days-spent30days).toFixed(2),
      (limit365days-spent365days).toFixed(2)
      ])
    }}€</h3>

    <h3>Spent according to statement</h3>
    <p>Last 1 day: {{spent1day.toFixed(2)}}€ / {{limit1day}}€</p>
    <p>Last 7 days: {{spent7days.toFixed(2)}}€ / {{limit7days}}€</p>
    <p>Last 30 days: {{spent30days.toFixed(2)}}€ / {{limit30days}}€</p>
    <p>Last 365 days: {{spent365days.toFixed(2)}}€ / {{limit365days}}€</p>

    <h3>Future spendable amounts</h3>
    
    <table style="width: 100%;">
      <thead>
        <tr>
          <th>Date</th>
          <th>Spent -24h</th>
          <th>Spent -7d</th>
          <th>Spent -30d</th>
          <th>Spent -365d</th>
          <th>Maximum spendable daily amount</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="d in daysArray" v-bind:key="d.day">
          <td style="border-bottom: 1px dashed #dedede;">{{d.dayFormatted}}</td>
          <td style="border-bottom: 1px dashed #dedede;">{{d.days1.toFixed(2)}}€ <strong><small>(~{{d.days1spendable.toFixed(2)}}€)</small></strong></td>
          <td style="border-bottom: 1px dashed #dedede;">{{d.days7.toFixed(2)}}€ <strong><small>(~{{d.days7spendable.toFixed(2)}}€)</small></strong></td>
          <td style="border-bottom: 1px dashed #dedede;">{{d.days30.toFixed(2)}}€ <strong><small>(~{{d.days30spendable.toFixed(2)}}€)</small></strong></td>
          <td style="border-bottom: 1px dashed #dedede;">{{d.days365.toFixed(2)}}€ <strong><small>(~{{d.days365spendable.toFixed(2)}}€)</small></strong></td>
          <td style="border-bottom: 1px dashed #dedede;">
            <strong>{{Math.min.apply(null, d.spendableArray).toFixed(2)}}€</strong>
          </td>
        </tr>
      </tbody>
    </table>
    
  </div>
</template>

<script>
import moment from 'moment'
import Papa from 'papaparse'

export default {
  name: 'HelloWorld',
  props: {
    msg: String
  },
  data() {
    return {
      days: {},
      daysArray: [],
      spent1day: 0,
      limit1day: 2300,
      spent7days: 0,
      limit7days: 2400,
      spent30days: 0,
      limit30days: 4000,
      spent365days: 0,
      limit365days: 50000,
      transactions: [],
      tier: 'starter',
      stake: 'none',
      file: '',
      content: [],
      parsed: false
    }
  },
  mounted() {
    
    let tier = localStorage.getItem('tier');
    if(tier != null) {
      this.tier = tier;
      this.changeTier();
    }
    let stake = localStorage.getItem('stake');
    if(stake != null) {
      this.stake = stake;
      this.changeStake();
    }

    let csv = JSON.parse(localStorage.getItem('csv'));
    if(csv != null) {
      this.calculateSpendable(csv);
      this.calculateFutureSpendable();
    }
   
  },
  methods: {
    changeStake () {
      localStorage.setItem('stake', this.stake);
    },
    changeTier () {
      if(this.tier == 'starter') {
        this.limit1day = 2300;
        this.limit7days = 2500;
        this.limit30days = 4000;
        this.limit365days = 50000;
      }
      if(this.tier == 'everyday') {
        this.limit1day = 5200;
        this.limit7days = 9100;
        this.limit30days = 14000;
        this.limit365days = 100000;
      }
      if(this.tier == 'premium') {
        this.limit1day = 7200;
        this.limit7days = 14100;
        this.limit30days = 20000;
        this.limit365days = 100000;
      }
      localStorage.setItem('tier', this.tier);
      let csv = JSON.parse(localStorage.getItem('csv'));
      if(csv != null) {
        this.calculateSpendable(csv);
        this.calculateFutureSpendable();
      }
    },
    handleFileUpload( event ){
        this.file = event.target.files[0];
        this.parseFile();
    },
    calculateSpendable(results) {

      let last1day = new Date();
      last1day.setHours(0, 0, 0);

      let last7days = new Date();
      last7days.setDate(last7days.getDate()-7);

      let last30days = new Date();
      last30days.setDate(last30days.getDate()-30);

      let last365days = new Date();
      last365days.setDate(last365days.getDate()-365);

      this.spent1day = 0;
      this.spent7days = 0;
      this.spent30days = 0;
      this.spent365days = 0;
      
      results.data.forEach((row) => {
        if(row.type == "PURCHASE" || row.type == "PENDING") {
          let date = new Date(row.date);

          if(date > last1day) {
            this.spent1day += (row.amount/100);
          }

          if(date > last7days) {
            this.spent7days += (row.amount/100);
          }

          if(date > last30days) {
            this.spent30days += (row.amount/100);
          }

          if(date > last365days) {
            this.spent365days += (row.amount/100);
          }

          this.transactions[date] = {"amount": (row.amount / 100), "description": row.description, "date": date};
        }
      });
    },
    calculateFutureSpendable() {
      this.daysArray = [];
      var today = new Date();
      for(var i=0; i<30; i++){
        let spent1day = 0;
        let spent7days = 0;
        let spent30days = 0;
        let spent365days = 0;
        var day = new Date(today.getFullYear(), today.getMonth(), today.getDate() + i, today.getHours(), today.getMinutes(), today.getSeconds());

        let daysAgo = {
            1: new Date(today.getFullYear(), today.getMonth(), today.getDate() + i, 0, 0, 0),
            7: new Date(today.getFullYear(), today.getMonth(), today.getDate() + i, today.getHours(), today.getMinutes(), today.getSeconds()),
            30: new Date(today.getFullYear(), today.getMonth(), today.getDate() + i, today.getHours(), today.getMinutes(), today.getSeconds()),
            365: new Date(today.getFullYear(), today.getMonth(), today.getDate() + i, today.getHours(), today.getMinutes(), today.getSeconds()),
          };
          
          //daysAgo[1].setDate(daysAgo[1].getDate());
          daysAgo[7].setDate(daysAgo[7].getDate()-7);
          daysAgo[30].setDate(daysAgo[30].getDate()-30);
          daysAgo[365].setDate(daysAgo[365].getDate()-365);
        
          
        
        for (const [txDate, row] of Object.entries(this.transactions)) {
          let transactionDate = new Date(txDate);
          if(transactionDate > daysAgo[1]) {
            spent1day += row.amount;
            console.log(daysAgo[1] + " - " + transactionDate + " - " +  row.amount);
          }
          if(transactionDate > daysAgo[7]) {
            spent7days += row.amount;
          }
          if(transactionDate > daysAgo[30]) {
            spent30days += row.amount;
          }
          if(transactionDate > daysAgo[365]) {
            spent365days += row.amount;
          }
        }

        this.days[day] = {
          day: day,
          dayFormatted: moment(String(day)).format('YYYY-MM-DD HH:mm:ss'),
          days1: spent1day,
          days7: spent7days,
          days30: spent30days,
          days365: spent365days,
          days1spendable: (this.limit1day - spent1day),
          days7spendable: (this.limit7days - spent7days),
          days30spendable: (this.limit30days - spent30days),
          days365spendable: (this.limit365days - spent365days),
          spendableArray: [parseFloat((this.limit1day - spent1day)), parseFloat(this.limit7days - spent7days), parseFloat(this.limit30days - spent30days), parseFloat(this.limit365days - spent365days)]
        };  
        
        this.daysArray.push(this.days[day]);
      }

      localStorage.setItem('CSVArray', JSON.stringify(this.daysArray));
        
    },
    parseFile(){
        Papa.parse( this.file, {
            header: true,
            skipEmptyLines: true,
            complete: function( results ){

              localStorage.setItem('csv', JSON.stringify(results));
              

              this.calculateSpendable(results);
              this.calculateFutureSpendable();

              //console.log(this.transactions);

              
              this.content = results;
              this.parsed = true;
            }.bind(this)
        } );
    },
  }
}
</script>

<style scoped>

.upload {
  width: 50%;
}
@media only screen and (max-width:801px) {
  .upload {
    width: 100%;
  }
  table {
    font-size: 0.8em;
  }
}

body {
  color: #203653;
}
h3 {
  margin: 40px 0 0;
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
</style>
