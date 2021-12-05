<script>
	import { data } from './example.js'
	import { onMount } from 'svelte';
	let plot;
	let DEPTH_VALUES = [1,5,10,15,20];
	let graph_data = [];
	let QUERY_INTERVAL = 10;
	for (let d in DEPTH_VALUES) {
		graph_data.push({
			x: [],
			y: [],
			type: 'scatter'
		});
	}
	
	function metric(evals) {
		return evals.reduce((a, b) => Math.max(a, b)) - evals.reduce((a, b) => a + b) / evals.length;
	}

	function parseData(obj) {
		let black = {
			x: [],
			y: [],
			type : 'scatter'
		};
		let white = {
			x : [],
			y : [],
			type : 'scatter'
		};

		for (let move in obj) {
			let move_number = Math.round(move / 2);
			let color;
			if (move % 2 == 0) {
				color = white;
			} else {
				color = black;
			}
			color.x.push(move_number);
			color.y.push(metric(obj[move].evals));
		}

		return [white, black];
	};

	function zip(a, b) {
		let i = 0;
		let res = [];
		while (i < a.length && i < b.length) {
			res.push([a[i], b[i]]);
			i+=1;
		}
		return res;
	}

    onMount(() => {
		let updateGraph = () => {
			let query_promises = [];
			for (let d of DEPTH_VALUES) {
				let url = 'http://127.0.0.1:5000/?depth=' + d;
				query_promises.push(
					fetch(url, {mode:'cors'}).then((r) => r.json())
				);
			}

			Promise.all(query_promises).then((query_results) => {
				let i = 0;
				graph_data = [];
				for (let group of zip(DEPTH_VALUES, query_results)) {
					let d = group[0];
					let result = group[1];
					for (let trace of parseData(result)) {
						graph_data.push(trace);
					}
					i += 1;
				}

				console.log(graph_data)
				Plotly.newPlot(plot.id, graph_data);
			});

		};

		Plotly.newPlot(plot.id, graph_data).then(
			() => {
				updateGraph();
				setInterval(updateGraph, QUERY_INTERVAL * 1000);
			}
		);


    });
	
</script>


<main class = "graph-panel">
	<div id="hahahah" bind:this={plot}>
	</div>
</main>


<style>
	main {
		display: flex;
		flex-direction: row;
	}
</style>