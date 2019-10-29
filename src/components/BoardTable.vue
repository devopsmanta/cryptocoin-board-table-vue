<template>
  <div class="board-wrapper">
    <div class="loader" v-if="isLoading">
      <div class="progress">
        <div class="indeterminate"></div>
      </div>
    </div>
    <div v-else id="cryptocoin-board-table">
      <div class="table-row">
        <div class="table-column">サイズ</div>
        <div class="table-column">サイズ</div>
        <div class="table-column">価格</div>
        <div class="table-column">%</div>
      </div>
      <div class="table-row" v-for="(val, index) in askData" :key="'ask' + index">
        <div class="table-column" style="align-items: center">
          <div
            :style="{
                width: Math.min(val / -20000, 100) + '%',
                height: '60%',
                backgroundColor: '#1C9FA0'}"
          ></div>
        </div>
        <div class="table-column">
          <div style="z-index: 1">{{getDigitsP(-val)}}</div>
        </div>
        <div class="table-column">
          <div style="z-index: 1">{{askDataLabel[index].toFixed(2)}}</div>
        </div>
        <div class="table-column">
          <div style="z-index: 1">{{getPercentage(askDataLabel[index], middle_price) + '%'}}</div>
        </div>
      </div>
      <div class="table-row">
        <div
          class="table-column"
          :style="{
                flexDirection: 'row',
                justifyContent: 'center',
                alignItems: 'center',
                color: tickDirection > 0 ? '#1C9FA0' : '#E54646',
                paddingRight: '25%',
                justifyContent: 'flex-end'}"
        >
          {{middle_price}}
          <i :class="'fa ' + gettickDirectionState()"></i>
        </div>
      </div>
      <div class="table-row" v-for="(val, index) in bidData" :key="'bid' + index">
        <div class="table-column" style="align-items: center">
          <div
            :style="{
                width: Math.min(val / -20000, 100) + '%',
                height: '60%',
                backgroundColor: '#E54646'}"
          ></div>
        </div>
        <div class="table-column">
          <div style="z-index: 1">{{getDigitsP(-val)}}</div>
        </div>
        <div class="table-column">
          <div style="z-index: 1">{{bidDataLabel[index].toFixed(2)}}</div>
        </div>
        <div class="table-column">
          <div style="z-index: 1">{{getPercentage(bidDataLabel[index], middle_price) + '%'}}</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: "BoardTable",
  data() {
    return {
      middle_price: "",
      tickDirection: 0,
      askData: [],
      askDataLabel: [],
      bidData: [],
      bidDataLabel: [],
      isLoading: true,
      isPrevState: false
    };
  },
  mounted() {
    let that = this;
    let wsBitMEXTrade = new WebSocket("wss://www.bitmex.com/realtime");
    let wsBitMEX = new WebSocket("wss://www.bitmex.com/realtime");
    wsBitMEXTrade.onopen = function() {
      wsBitMEXTrade.send(
        JSON.stringify({ op: "subscribe", args: ["trade:XBTUSD"] })
      );
    };

    wsBitMEXTrade.onmessage = function(msg) {
      let response = JSON.parse(msg.data);
      if (response.data) {
        let middle_price = response.data[0].price.toFixed(2);
        let tickDirection;
        if (!that.isLoading) {
          switch (response.data[0].tickDirection) {
            case "PlusTick":
            case "ZeroPlusTick":
              tickDirection = 1;
              break;
            case "MinusTick":
            case "ZeroMinusTick":
              tickDirection = -1;
              break;
            default:
              tickDirection = 0;
              break;
          }
        }
        if (that.isPrevState) {
          for (let i = 0; i < that.askData.length; i++) {
            let askPercentageChange =
              that.getPercentage(that.askDataLabel[i], middle_price) -
              that.getPercentage(that.askDataLabel[i], that.middle_price);
            let bidPercentageChange =
              that.getPercentage(that.bidDataLabel[i], middle_price) -
              that.getPercentage(that.bidDataLabel[i], that.middle_price);
            if (Math.abs(askPercentageChange) >= 0.01)
              that.insertFlashElement(i + 1, 3, askPercentageChange);
            if (Math.abs(bidPercentageChange) >= 0.01)
              that.insertFlashElement(i + 12, 3, bidPercentageChange);
          }
        }
        that.middle_price = middle_price;
        that.tickDirection = tickDirection;
      }
    };

    wsBitMEX.onopen = function() {
      wsBitMEX.send(
        JSON.stringify({ op: "subscribe", args: ["orderBook10:XBTUSD"] })
      );
    };

    wsBitMEX.onmessage = function(msg) {
      var response = JSON.parse(msg.data);
      if (response.data) {
        let askDataLabel = response.data[0].asks.map(x => x[0]);
        let askData = response.data[0].asks.map(x => -x[1]);
        askDataLabel.reverse();
        askData.reverse();
        let bidDataLabel = response.data[0].bids.map(x => x[0]);
        let bidData = response.data[0].bids.map(x => -x[1]);
        if (that.isLoading) {
          that.isLoading = false;
        } else {
          that.isPrevState = true;
          for (let i = 0; i < askData.length; i++) {
            let askPriceChange = askDataLabel[i] - that.askDataLabel[i];
            let askSizeChange = -(askData[i] - that.askData[i]);
            let askPercentageChange =
              that.getPercentage(askDataLabel[i]) -
              that.getPercentage(that.askDataLabel[i]);
            let bidPriceChange = bidDataLabel[i] - that.bidDataLabel[i];
            let bidSizeChange = -(bidData[i] - that.bidData[i]);
            let bidPercentageChange =
              that.getPercentage(bidDataLabel[i]) -
              that.getPercentage(that.bidDataLabel[i]);
            that.insertFlashElement(i + 1, 1, askSizeChange);
            that.insertFlashElement(i + 1, 2, askPriceChange);
            if (Math.abs(askPercentageChange) >= 0.01)
              that.insertFlashElement(i + 1, 3, askPercentageChange);
            that.insertFlashElement(i + 12, 1, bidSizeChange);
            that.insertFlashElement(i + 12, 2, bidPriceChange);
            if (Math.abs(bidPercentageChange) >= 0.01)
              that.insertFlashElement(i + 12, 3, bidPercentageChange);
          }
        }
        that.askData = askData;
        that.askDataLabel = askDataLabel;
        that.bidData = bidData;
        that.bidDataLabel = bidDataLabel;
      }
    };
  },
  methods: {
    getDigitsP: function(digits) {
      let digLen = digits.toString().length;
      switch (digLen) {
        case 5:
          return `${digits}          `;
        case 6:
          return `${digits}        `;
        case 7:
          return `${digits}      `;
        case 8:
          return `${digits}    `;
        case 9:
          return `${digits}  `;
        case 10:
          return `${digits}`;
        default:
          return `${digits}`;
      }
    },
    gettickDirectionState: function() {
      switch (this.tickDirection) {
        case 1:
          return "fa-caret-up";
        case -1:
          return "fa-caret-down";
        default:
          return "";
      }
    },
    getPercentage: function(price, middle_price) {
      return (((price - middle_price) / middle_price) * 100).toFixed(2);
    },
    insertFlashElement(row, col, change) {
      if (change === 0) return;
      let rowElement = document.getElementById("cryptocoin-board-table")
        .children[row];
      if (rowElement === null) return;
      let element = rowElement.children[col];
      if (element === null) return;
      let flashyElement = document.createElement("div");
      let backgroundColor = change > 0 ? "#1C9FA0" : "#E54646";
      flashyElement.style = `background-color: ${backgroundColor}; width: 100%; height: 100%; position: absolute`;
      element.insertBefore(flashyElement, element.firstChild);
      setTimeout(
        tag => {
          element.removeChild(tag);
        },
        100,
        flashyElement
      );
    }
  }
};
</script>

<style>
* {
  box-sizing: border-box;
}

#loader {
  z-index: 1;
}

#cryptocoin-board-table {
  display: flex;
  flex-direction: column;
  border: 1px solid rgb(49, 49, 52);
  width: 100%;
  height: 100vh;
  border-collapse: collapse;
  text-align: right;
  font-size: 2vh !important;
  table-layout: fixed;
  margin: 0 auto !important;
  color: white;
}

.table-row {
  flex: 1;
  display: flex;
  flex-direction: row;
  flex-wrap: wrap;
  width: 100%;
}

#cryptocoin-board-table > .table-row:nth-child(odd) {
  background-color: rgb(38, 38, 38);
}
#cryptocoin-board-table > .table-row:nth-child(even) {
  background-color: rgb(20, 24, 34);
}
.table-column {
  display: flex;
  flex-basis: 100%;
  flex-flow: row-reverse;
  flex: 1;
  border: 1px solid rgb(49, 49, 52);
  margin-top: -1px;
  margin-left: -1px;
  align-items: center;
  position: relative;
}
.loader {
  position: absolute;
  margin: auto;
  top: 50vh;
  bottom: 50vh;
  left: 0;
  right: 0;
}
.board-wrapper {
  width: 100%;
  height: 100vh;
  background-color: rgb(20, 24, 34);
}
</style>