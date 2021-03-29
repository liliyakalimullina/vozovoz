<template>
  <div class="calculator">
    <div class="calculator-options">
      <h3>Рассчитать</h3>
      <div class="calculator-row">
        <div class="calculator-options-item">
          <client-only>
            <v-select :options="locations" v-model="dispatchLocation" @search="searchLocation" class="calculator-select"></v-select>
          </client-only>
          <div class="calculator-radio">
            <input name="option-from" type="radio" id="address-from" value="address" v-model="dispatchLocationPoint"> 
            <label for="address-from">От адреса</label> 
            <input name="option-from" type="radio" id="terminal-from" value="terminal" v-model="dispatchLocationPoint" :disabled="dispatchLocation && !dispatchLocation.hasTerminal"> 
            <label for="terminal-from">От терминала</label>
          </div>
        </div>
        <div class="calculator-options-icon">
        </div>
        <div class="calculator-options-item">
          <client-only>
            <v-select :options="locations" v-model="destinationLocation" @search="searchLocation" class="calculator-select"></v-select>
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
          <div class="calculator-options-item-title">Вес, кг</div>
          <input type="text" @change="calculatePrice()" v-model="weight" v-mask="'#####.#'" class="calculator-options-item-input">
        </div>
        <div class="calculator-options-item">
          <div class="calculator-options-item-title">Объем, м<sup>3</sup></div>
          <input type="text"  @change="calculatePrice()" v-model="volume" v-mask="'##.##'" class="calculator-options-item-input">
        </div>
      </div>
    </div>
    <div class="calculator-price">
      <div class="calculator-price-delivery">
        Доставка: <span>{{ delivery }}</span>
      </div>
      <div class="calculator-price-amount">
        {{ price }} ₽<sup>*</sup>
      </div>
      <div class="calculator-price-base-amount">
        {{ basePrice }} ₽
      </div>
      <div class="calculator-price-description">
        *При заказе через личный кабинет и максимальных параметрах одного места: длина – 0.2 м, ширина – 0.2 м, высота – 0.2 м и вес – 2.5 кг
      </div>
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
      volume: '00.1',
      weight: '00000.9',
      dispatchLocation: { code:'e90f1820-0128-11e5-80c7-00155d903d03', label: 'Москва', hasTerminal: true },
      dispatchLocationPoint: 'terminal', 
      destinationLocation: { code: 'e90f19de-0128-11e5-80c7-00155d903d03', label: 'Санкт-Петербург', hasTerminal: true },
      destinationLocationPoint: 'terminal'
    }
  },
  created() {
    this.$axios.post(this.url, {
      object: 'location',
      action: 'get',
      params: {
        limit: 100,
        offset: 0
      }
    })
    .then(response => {
      console.log(response)
      const responseData: any[]  = response.data.response.data
      for(let i = 0; i < responseData.length; i++) {
        this.locations.push({ label: responseData[i].name, code: responseData[i].guid, hasTerminal: !!responseData[i].default_terminal })
      }
      this.calculatePrice()
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
    searchLocation (search: string, loading: any): void {
      console.log(search + ' ' + loading)
      this.$axios.post(this.url, {
        object: 'location',
        action: 'get',
        params: {
          search: search,
        }
      })
      .then(response => {
        const responseData: any[]  = response.data.response.data
        this.locations.length = 0
        for(let i = 0; i < responseData.length; i++) {
          this.locations.push({ label: responseData[i].name, code: responseData[i].guid, hasTerminal: !!responseData[i].default_terminal })
        }
        console.log(this.locations)
      })
    }, 
    calculatePrice(): void {
      this.$axios.post(this.url, {
        object: 'price',
        action: 'get',
        params: {
          cargo : {
            dimension: {
              quantity: 1,
              volume: this.volume,
              weight: this.weight
            }
          },
          gateway: {
            dispatch: {
              point: {
                location: this.dispatchLocation.code,
                terminal: (this.dispatchLocationPoint === 'terminal' ? 'default' : '')
              }
            },
            destination: {
              point: {
                location: this.destinationLocation.code,
                terminal: (this.destinationLocationPoint === 'terminal' ? 'default' : '')
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
      else {
        this.calculatePrice()
      }
    },
    destinationLocation: function() {
      if(this.destinationLocation && !this.destinationLocation.hasTerminal){
        this.destinationLocationPoint = 'address'
      }
      else {
        this.calculatePrice()
      }
    },
    dispatchLocationPoint: function() {
      this.calculatePrice()
    },
    destinationLocationPoint: function() {
      this.calculatePrice()
    }
  }
})
</script>

<style lang='scss'>
.calculator {
  border: 1px solid #ccc;
  max-width: 800px;
  width: 100%;
  margin: 70px auto 0;
  display: flex;
  justify-content: space-between;
  padding: 30px;
  box-shadow: 0 0 10px rgba(199,199,199,0.7);
  border-radius: 5px;
  h3 {
    font-size: 24px;
    font-weight: 400;
    margin-top: 0px;
    margin-bottom: 0px;
  }
  &-row {
    display: flex;
    justify-content: space-between;
  }
  &-select {
    color: #ccc;
    width: 220px;
    margin-bottom: 15px;
  }
  &-options {
    max-width: 500px;
    width: 100%;
    &-item {
      margin-top: 40px;
      &-title {
        margin-bottom: 15px;
      }
      &-input {
        width: 200px;
        padding: 8px 10px;
        border: 1px solid #ccc;
        border-radius: 5px;
        &:focus {
          border: 1px solid red !important;
          outline: none;
        }
      }
    }
  }
  &-price {
    max-width: 240px;
    width: 100%;
    text-align: center;
    &-base-amount {
      color: #8b8b8b;
      font-size: 35px;
      font-weight: 300;
      position: relative;
      width: 140px;
      margin: 0 auto;
      &::before {
        content: "";
        position: absolute;
        width: 100%;
        height: 2px;
        top: 50%;
        left: 0;
        transform: rotate(-7deg);
        z-index: 1;
        box-shadow: 0 0 1px 1px #fff;
        background-color: #8b8b8b;
      }
    }
    &-amount {
      color: #ec1b22;
      font-weight: 700;
      font-size: 50px;
      margin-top: 30px;
      margin-bottom: 10px;
    }
    &-description {
      font-size: 12px;
      font-weight: 300;
      max-width: 227px;
      width: 100%;
      margin: 10px auto 0;
    }
    &-delivery {
      margin-top: 5px;
      span {
        color: #ec1b22;
      }
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
