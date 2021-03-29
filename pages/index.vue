<template>
  <div class="calculator">
    <div class="calculator-options">
      <h3>Рассчитать</h3>
      <div class="calculator-row">
        <div class="calculator-options-item">
          <client-only>
            <v-select :options="locations" v-model="dispatchLocation" @search="searchLocation"  ></v-select>
          </client-only>
          <div class="calculator-radio">
            <input name="option-from" type="radio" id="address-from" value="address" v-model="dispatchLocationPoint" > 
            <label for="address-from">От адреса</label> 
            <input name="option-from" type="radio" id="terminal-from" value="terminal" v-model="dispatchLocationPoint" :disabled="dispatchLocation && !dispatchLocation.hasTerminal"> 
            <label for="terminal-from">От терминала</label>
          </div>
        </div>
        <div class="calculator-options-icon">
        </div>
        <div class="calculator-options-item">
          <client-only>
            <v-select :options="locations" v-model="destinationLocation" @search="searchLocation" ></v-select>
          </client-only>
          <div class="calculator-radio">
            <input name="option-to" type="radio" id="address-to" value="address" v-model="destinationLocationPoint" :checked="destinationLocation && !destinationLocation.hasTerminal"> 
            <label for="address-to">До адреса</label> 
            <input name="option-to" type="radio" id="terminal-to" value="terminal" v-model="destinationLocationPoint" :disabled="destinationLocation && !destinationLocation.hasTerminal"> 
            <label for="terminal-to">До терминала</label>
          </div>
        </div>
      </div>
      <div class="calculator-row">
        <div class="calculator-options-item">
          <div>Вес</div>
          <input type="text" v-model="weight">
        </div>
        <div class="calculator-options-item">
          <div>Объем</div>
          <input type="text" v-model="volume">
        </div>
      </div>
    </div>
    <div class="calculator-price">
      <div class="calculator-price-delivery">
        Доставка: {{ delivery }}
      </div>
      <div class="calculator-price-amount">
        {{ price }} Р
      </div>
      <div class="calculator-price-old-amount">
        {{ basePrice }} P
      </div>
      <div class="calculator-price-description">
        *При заказе через личный кабинет и максимальных параметрах одного места: длина – 0.2 м, ширина – 0.2 м, высота – 0.2 м и вес – 2.5 кг
      </div>
      <button @click="send">Оформить заказ</button>
    </div>
  </div>
</template>

<script lang="ts">
import Vue from 'vue'

export default Vue.extend({
  data() {
    return {
      url: 'https://vozovoz.ru/api/?token=K5ZCHajHiE4L8WJs7WXo4aSBmmd5GTNZ62EmwoOH',
      locations: Array(),
      basePrice: 0,
      price: 0,
      deliveryTimeFrom: 1,
      deliveryTimeTo: 1,
      volume: '0.1',
      weight: '0.9',
      dispatchLocation: { code:'e90f1820-0128-11e5-80c7-00155d903d03', label: 'Москва, Город федерального значения', hasTerminal: true },
      dispatchLocationPoint: 'terminal', 
      destinationLocation: { code: 'e90f19de-0128-11e5-80c7-00155d903d03', label: 'Санкт-Петербург, Город федерального значения', hasTerminal: true },
      destinationLocationPoint: 'terminal',
      optionsChanged: false
    }
  },
  created() {
    this.$axios.post(this.url, {
      "object": "location",
      "action": "get",
      "params": {
        limit: 100,
        offset: 0
      }
    })
    .then(response => {
      console.log(response)
      const responseData: any[]  = response.data.response.data
      for(let i = 0; i < responseData.length; i++) {
        this.locations.push({ label: responseData[i].name + ', ' + responseData[i].region_str, code: responseData[i].guid, hasTerminal: !!responseData[i].default_terminal })
      }
      this.send()
    })
  },
  computed: {
    delivery: function(): string {
      if(this.deliveryTimeFrom === this.deliveryTimeTo  && this.deliveryTimeFrom == 1) {
        return 'на следующий день'
      }
      else if(this.deliveryTimeFrom === this.deliveryTimeTo) {
        return this.deliveryTimeFrom  + ' дня' 
      }
      else {
        return 'от ' + this.deliveryTimeFrom + ' до ' + this.deliveryTimeTo + '  дней'
      }
    }
  },
  methods: {
    searchLocation (search: string, loading: any) {
      console.log(search + ' ' + loading)
      this.getLocations(search)
    },
    getLocations(search: string) {
      this.$axios.post(this.url, {
        "object": "location",
        "action": "get",
        "params": {
          search: search,
        }
      })
      .then(response => {
        const responseData: any[]  = response.data.response.data
        this.locations.length = 0
        for(let i = 0; i < responseData.length; i++) {
          this.locations.push({ label: responseData[i].name + ', ' + responseData[i].region_str, code: responseData[i].guid, hasTerminal: !!responseData[i].default_terminal })
        }
        console.log(this.locations)
      })
    }, 
    getDispatchPoint(): string {
      return this.dispatchLocationPoint
    },
    getDestinationPoint(): string {
      return this.destinationLocationPoint
    },
    send(): void {

      this.$axios.post(this.url, {
        "object": "price",
        "action": "get",
        "params": {
          "cargo" : {
            "dimension": {
              "quantity": 1,
              "volume": this.volume,
              "weight": this.weight
            }
          },
          "gateway": {
            "dispatch": {
              "point": {
                "location": this.dispatchLocation.code,
                "terminal": (this.dispatchLocationPoint === 'terminal' ? 'default' : '')
              }
            },
            "destination": {
              "point": {
                "location": this.destinationLocation.code,
                "terminal": (this.destinationLocationPoint === 'terminal' ? 'default' : '')
              }
            }
          }
        }
      })
      .then(response => {
        console.log(response)
        const responseData = response.data.response
        this.basePrice = responseData.basePrice
        this.price = responseData.price
        this.deliveryTimeFrom = responseData.deliveryTime.from
        this.deliveryTimeTo = responseData.deliveryTime.to       
      })
    }
  },
  watch: {
    dispatchLocation: function() {
      if(this.dispatchLocation && !this.dispatchLocation.hasTerminal){
        this.dispatchLocationPoint = 'address'
      }
      this.send()
    },
    destinationLocation: function() {
      if(this.destinationLocation && !this.destinationLocation.hasTerminal){
        this.destinationLocationPoint = 'address'
      }
      this.send
    },
    dispatchLocationPoint: function() {
      this.send()
    },
    destinationLocationPoint: function() {
      this.send()
    },
    volume: function() {
      this.send()
    },
    weight: function() {
      this.send()
    }
  }
})
</script>

<style lang='scss'>
.calculator {
  border: 1px solid #ccc;
  max-width: 800px;
  width: 100%;
  margin: 0 auto;
  display: flex;
  justify-content: space-between;
  padding: 20px;
  &-row {
    display: flex;
    justify-content: space-between;
  }
  &-select {
    color: #ccc;
  }
  &-options {
    max-width: 500px;
    width: 100%;
  }
  &-price {
    max-width: 250px;
    width: 100%;
    &-amount {
      color: #ec1b22;
      font-weight: 700;
      font-size: 50px;
    }
    &-description {
      font-size: 12px;
      font-weight: 300;
    }
  }
  label {
    font-size: 14px;
  }
  .vs__clear{
    display: none;
  }
}
</style>
