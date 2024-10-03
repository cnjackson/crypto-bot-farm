<template>
  <v-container>
    <v-row>
      <v-col>
        <h1 class="text-center">Crypto Trading Bot Simulator</h1>
      </v-col>
    </v-row>

    <!-- Input: Set max pairs -->
    <v-row class="mt-4">
      <v-col cols="12" sm="6">
        <v-text-field
          label="Set Maximum Pairs"
          v-model="maxPairs"
          type="number"
          min="1"
          outlined
        />
      </v-col>
    </v-row>

    <!-- Add/Remove Pairs Section -->
    <v-row v-for="(pair, index) in tradingPairs" :key="index" class="mt-3">
      <v-col cols="12">
        <trading-pair-form
          :pair-index="index"
          :pair="pair"
          :available-pairs="availablePairs"
          @update-pair="updatePair"
          @remove-pair="removePair(index)"
        />
      </v-col>
    </v-row>

    <!-- Add Pair Button -->
    <v-row>
      <v-col>
        <v-btn
          class="my-4"
          color="primary"
          @click="addPair"
          :disabled="tradingPairs.length >= maxPairs"
        >
          Add Pair
        </v-btn>
      </v-col>
    </v-row>

    <!-- Run Simulation Button -->
    <v-row>
      <v-col>
        <v-btn color="success" class="my-4" @click="runSimulation">
          Run Simulation
        </v-btn>
      </v-col>
    </v-row>

    <!-- Results Section -->
    <v-row v-if="simulationResults.length > 0">
      <v-col cols="12" v-for="(result, index) in simulationResults" :key="index">
        <v-card class="pa-4 mb-4">
          <p><strong>{{ result.pair }}</strong></p>
          <p><strong>Viva Bot P&L:</strong> BTC: {{ result.vivaBTC.toFixed(4) }}, USD: {{ result.vivaUSD.toFixed(2) }}</p>
          <p><strong>Piggyback Bot P&L:</strong> BTC: {{ result.piggybackBTC.toFixed(4) }}, USD: {{ result.piggybackUSD.toFixed(2) }}</p>
          <v-divider></v-divider>
          <v-card-actions>
            <canvas :id="'balanceChart-' + index"></canvas>
          </v-card-actions>
        </v-card>
      </v-col>
    </v-row>
  </v-container>
</template>

<script>
import TradingPairForm from './components/TradingPairForm.vue';
import { Chart, registerables } from 'chart.js';
Chart.register(...registerables);

export default {
  name: 'App',
  components: {
    TradingPairForm,
  },
  data() {
    return {
      maxPairs: 3, // Default max pairs, can be changed
      tradingPairs: [], // List of added pairs
      availablePairs: [], // This will be populated dynamically
      simulationResults: [],
    };
  },
  async created() {
    // Fetch available spot pairs from Binance when the app is loaded
    await this.fetchAvailableSpotPairs();
  },
  methods: {
    async fetchAvailableSpotPairs() {
      const endpoint = `https://api.binance.com/api/v3/exchangeInfo`;
      try {
        const response = await fetch(endpoint);
        const data = await response.json();

        // Filter for spot trading pairs where base asset is BTC and sort them alphabetically by baseAsset
        this.availablePairs = data.symbols
          .filter((symbol) => symbol.symbol.endsWith('BTC')) // Ensure the quote asset is BTC
          .filter((symbol) => symbol.isSpotTradingAllowed) // Ensure it is available on the spot market
          .map((symbol) => ({
            symbol: symbol.symbol, // Full symbol (e.g., SOLBTC)
            baseAsset: symbol.baseAsset,
            quoteAsset: symbol.quoteAsset,
          }))
          .sort((a, b) => a.baseAsset.localeCompare(b.baseAsset)); // Sort by baseAsset (e.g., ETH, SOL, XMR)
      } catch (error) {
        console.error('Error fetching available spot pairs from Binance:', error);
      }
    },
    addPair() {
      if (this.tradingPairs.length < this.maxPairs) {
        // Default to first available pair if the list is populated
        const defaultPair =
          this.availablePairs.length > 0 ? this.availablePairs[0].symbol : '';
        this.tradingPairs.push({
          pair: defaultPair, // Default to first available pair
          initialBTC: 0.1, // Default BTC per bot
        });
      }
    },
    updatePair(index, updatedPair) {
      // Replace the trading pair at the given index with the updated values
      this.tradingPairs.splice(index, 1, updatedPair);
    },
    removePair(index) {
      this.tradingPairs.splice(index, 1);
    },
    async runSimulation() {
      this.simulationResults = []; // Clear old results
      for (const pair of this.tradingPairs) {
        const result = await this.simulatePair(pair.pair, pair.initialBTC);
        this.simulationResults.push(result);
      }
      this.$nextTick(() => {
        this.simulationResults.forEach((result, index) => {
          this.createChart(index, result.btcBalanceOverTime);
        });
      });
    },
    async simulatePair(pair, initialBTC) {
      const prices = await this.fetchPriceData(pair);
      let vivaBTC = initialBTC;
      let piggybackBTC = initialBTC;
      let vivaUSD = 0;
      let piggybackUSD = 0;
      let btcBalanceOverTime = [initialBTC];

      // Viva bot logic: Sell at 5% profit and reinvest
      prices.forEach((price, i) => {
        if (i < prices.length - 1) {
          const { btcProfit, usdProfit } = this.calculateProfit(vivaBTC, price);
          vivaUSD += usdProfit;
          vivaBTC -= btcProfit; // Simulate selling
          btcBalanceOverTime.push(vivaBTC); // Track balance over time
        }
      });

      // Piggyback bot logic: DCA on dips, no selling
      prices.forEach((price, i) => {
        if (i < prices.length - 1 && price < prices[0] * 0.95) {
          const dcaAmount = initialBTC * 0.1; // Buy 10% of BTC on dips
          piggybackBTC += dcaAmount;
          btcBalanceOverTime.push(piggybackBTC); // Track DCA over time
        }
      });

      return {
        vivaBTC,
        vivaUSD,
        piggybackBTC,
        piggybackUSD,
        btcBalanceOverTime,
        pair,
      };
    },
    async fetchPriceData(pair) {
      const endpoint = `https://api.binance.com/api/v3/klines?symbol=${pair}&interval=1h&limit=500`;
      try {
        const response = await fetch(endpoint);
        const data = await response.json();
        return data.map((candle) => parseFloat(candle[4])); // Close price is at index 4
      } catch (error) {
        console.error('Error fetching price data from Binance:', error);
        return [];
      }
    },
    calculateProfit(btcAmount, entryPrice) {
      const profitTarget = entryPrice * 1.05; // 5% profit target
      const btcProfit = btcAmount * 0.05; // 5% of BTC amount
      const usdProfit = btcProfit * profitTarget;
      return { btcProfit, usdProfit, exitPrice: profitTarget };
    },
    createChart(index, balanceData) {
      const ctx = document
        .getElementById(`balanceChart-${index}`)
        .getContext('2d');
      new Chart(ctx, {
        type: 'line',
        data: {
          labels: Array(balanceData.length)
            .fill('')
            .map((_, i) => `Hour ${i + 1}`),
          datasets: [
            {
              label: 'BTC Balance Over Time',
              data: balanceData,
              borderColor: 'rgba(75, 192, 192, 1)',
              fill: false,
            },
          ],
        },
      });
    },
  },
};
</script>

<style scoped>
.text-center {
  text-align: center;
}
</style>
