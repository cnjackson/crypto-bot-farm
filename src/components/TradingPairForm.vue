<template>
	<div class="card mb-3">
		<div class="card-body">
			<h5>Trading Pair {{ pairIndex + 1 }}</h5>
			<div class="mb-3">
				<label for="crypto-pair" class="form-label">Select Trading Pair:</label>
				<select class="form-select" :value="pair.pair" @change="updatePair($event.target.value)">
					<option v-for="availablePair in availablePairs" :key="availablePair.symbol" :value="availablePair.symbol">{{ availablePair.baseAsset }}/{{ availablePair.quoteAsset }}</option>
				</select>
			</div>
			<div class="mb-3">
				<label for="initial-btc" class="form-label">Initial BTC per Bot:</label>
				<input type="number" class="form-control" :value="pair.initialBTC" step="0.01" min="0.01" @input="updateInitialBTC($event.target.value)" />
			</div>
			<button @click="removePair" class="btn btn-danger">Remove Pair</button>
		</div>
	</div>
</template>

<script>
export default {
	props: {
		pair: Object,
		availablePairs: Array,
		pairIndex: Number,
	},
	methods: {
		updatePair(newPair) {
			this.$emit("update-pair", this.pairIndex, { ...this.pair, pair: newPair });
		},
		updateInitialBTC(newBTC) {
			this.$emit("update-pair", this.pairIndex, { ...this.pair, initialBTC: parseFloat(newBTC) });
		},
		removePair() {
			this.$emit("remove-pair");
		},
	},
};
</script>

<style scoped>
.card {
	border: 1px solid #ddd;
}
</style>
