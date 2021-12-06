<script>
    //import {Chessboard} from "cm-chessboard"
	import Chart from 'chart.js/auto';
	import { data } from './example.js'
	import { onMount } from 'svelte';

	let whiteGraph, blackGraph;
	let DEPTH_VALUES = [1,5,10,15,20];
	let QUERY_INTERVAL = 10;
	let URL_ROOT = "https://sharp-api-qkzpapi3la-ue.a.run.app/sharpness";

	let FENS = [];
	let board;


	let COLORS = [
		'#377eb8',
		'#4daf4a',
		'#e41a1c',
		'#984ea3',
		'#ff7f00',
		'black'
	]

	function legendClickHandler(e, legendItem, legend) {
		let index = legendItem.datasetIndex;
		console.log(e);
		console.log(legendItem);
		console.log(legend);

		let ci = legend.chart;
		[
			ci.getDatasetMeta(index-1),
			ci.getDatasetMeta(index),
			ci.getDatasetMeta(index+1)
		].forEach(function(meta) {
			meta.hidden = meta.hidden === null ? !ci.data.datasets[index].hidden : null;
		});
		ci.update();
	};


	let whiteData = {
		labels : [],
		datasets : []
	};
	let blackData = {
		labels : [],
		datasets : []
	};
	
	function metric(evals) {
		return evals.reduce((a, b) => Math.max(a, b)) - evals.reduce((a, b) => a + b) / evals.length;
	}

	function parseData(obj) {
		let white_sharpness = [];
		let black_sharpness = [];
		let white_eval = [];
		let black_eval = [];
		let fens = [];
		let move = 0;
		while (obj[move]) {
			let evals = obj[move].evals;
			fens.push(obj[move].fen);
			if (move % 2 == 0) {
				white_sharpness.push(metric(evals));
				white_eval.push(Math.max(...evals));
			} else {
				black_sharpness.push(metric(evals));
				black_eval.push(Math.max(...evals));
			}
			
			move += 1;
		}
		return [white_sharpness, black_sharpness, white_eval, black_eval, fens];
	};

    onMount(() => {
		let whiteChartOptions = {
			elements: {
				point:{
					radius: 0
				}
			},
			borderColor: COLORS,
			responsive: true,
			plugins: {
				borderColor: COLORS,
				title: {
					display: true,
					text: 'White Sharpness'
				},
				legend: {
					position: 'right'
				}
			},
		};
		let whiteChart = new Chart(whiteGraph, {
			type: 'line',
			data: whiteData,
			options: whiteChartOptions,
		});
		let blackChartOptions = JSON.parse(JSON.stringify(whiteChartOptions));
		blackChartOptions.plugins.title.text = 'Black Sharpness';
		
		let blackChart = new Chart(blackGraph, {
			type: 'line',
			data: blackData,
			options: blackChartOptions,
		});


		let updateGraph = () => {
			let query_promises = [];
			for (let d of DEPTH_VALUES) {
				let url = URL_ROOT + `?game=${6}&depth=${d}`;
				query_promises.push(
					fetch(url)
						.then(
							r => {
								if (r.body) {
									return r.json();
								}
								return {};
							})
				);
			}

			Promise.all(query_promises).then((query_results) => {
				whiteData.datasets = []
				blackData.datasets = []
				
				let whiteEvaluation = [];
				let blackEvaluation = [];
				for (let j = 0; j < query_results.length; j++) {
					let result = query_results[j];
					let [white_metric, black_metric, white_eval, black_eval, fens] = parseData(result);
					whiteEvaluation = white_eval;
					blackEvaluation = black_eval;
					FENS = fens;
					whiteData.labels = [];
					blackData.labels = [];
					for (let i = 0; i < white_metric.length; i++) {
						whiteData.labels.push(i);
						blackData.labels.push(i);
					}
					
					whiteData.datasets.push({
						data: white_metric,
						tension: 0.4,
						label: `Depth ${DEPTH_VALUES[j]}`,
						hidden: j < DEPTH_VALUES.length-1,
						order: 10 - j
					});
					blackData.datasets.push({
						data: black_metric,
						tension: 0.4,
						label: `Depth ${DEPTH_VALUES[j]}`,
						hidden: j < DEPTH_VALUES.length-1,
						order: 10 - j
					});
				}
				
				whiteData.datasets.push({
					data: whiteEvaluation,
					tension: 0,
					label: `Evaluation`,
					hidden: true,
      				borderDash: [2, 2],
					order: -10
				});
				blackData.datasets.push({
					data: blackEvaluation,
					tension: 0,
					label: `Evaluation`,
					hidden: true,
      				borderDash: [2, 2],
					order: -10
				});

				
				//new Chessboard(board, {position: FENS[FENS.length - 1]});
				whiteChart.update();
				blackChart.update();
			});
		};
		updateGraph();
		// 	//setInterval(updateGraph, QUERY_INTERVAL * 1000);
    });
	
</script>


<main>
	<graph-panel class = "graph-panel">
		<div>
			<canvas id="white-graph" bind:this={whiteGraph}>
			</canvas>
		</div>
		<br/>
		<div>
			<canvas id="black-graph" bind:this={blackGraph}>
			</canvas>
		</div>
	</graph-panel>

	<div>
		<board bind:this={board}></board>
	</div>
</main>

<style>
	graph-panel {
		display: flex;
		flex-direction: column;
	}
	graph-panel > div {
		max-width: 600px;
	}
</style>