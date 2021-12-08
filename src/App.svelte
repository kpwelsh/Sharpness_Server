<script>
	import Chessboard from "chessboardjs";
	import Chart from 'chart.js/auto';
	import { data } from './example.js';
	import { onMount } from 'svelte';

	let whiteGraph, blackGraph;
	let DEPTH_VALUES = [1,5,10,15,20];
	let QUERY_INTERVAL = 10;
	let URL_ROOT = "https://sharp-api-qkzpapi3la-ue.a.run.app";

	let FENS = [];
	let fen_index = 0;
	let board;

	let game_options = [];
	let game_number;

	let whiteChart, blackChart;

	let COLORS = [
		'#377eb8',
		'#4daf4a',
		'#e41a1c',
		'#984ea3',
		'#ff7f00',
		'black'
	]

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

	function updateBoard(fen) {
		if (board === undefined){
			return;
		}
		while (board.firstChild) {
			board.removeChild(board.firstChild);
		}
		Chessboard(board.id, fen);
	};

	function updateGraph(game_number) {
		console.log(game_number)
		if (game_number === undefined || whiteChart === undefined || blackChart === undefined) {
			return;
		}
		let query_promises = [];
		for (let d of DEPTH_VALUES) {
			query_promises.push(
				fetch(`${URL_ROOT}/sharpness?game=${game_number}&depth=${d}`)
					.then(r => {
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

			
			fen_index = FENS.length - 1;
			updateBoard(FENS[fen_index]);
			whiteChart.update();
			blackChart.update();
		});
	};

	function onKeyDown(e) {
		console.log(e)
		if (e.keyCode == 37) {
			if (fen_index > 0) {
				fen_index = fen_index - 1;
			}
		} else if (e.keyCode == 39) {
			if (fen_index < FENS.length) {
				fen_index = fen_index + 1;
			}
		}
	};
	
	$: updateGraph(game_number);
	$: updateBoard(FENS[fen_index]);

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
		whiteChart = new Chart(whiteGraph, {
			type: 'line',
			data: whiteData,
			options: whiteChartOptions,
		});
		let blackChartOptions = JSON.parse(JSON.stringify(whiteChartOptions));
		blackChartOptions.plugins.title.text = 'Black Sharpness';
		
		blackChart = new Chart(blackGraph, {
			type: 'line',
			data: blackData,
			options: blackChartOptions,
		});


		fetch(`${URL_ROOT}/games`).then(async (r) => {
			game_options = (await r.json()).sort((a,b) => a-b);
			game_number = game_options[game_options.length - 1];
		});
	});
	
</script>

<svelte:window on:keydown={onKeyDown}/>
<main>
	<div class="row">
		<label for="Game">WCC 2021 Game:</label>
		<select name="Game" bind:value={game_number}>
			{#each game_options as game}
				<option value="{game}">{game}</option>
			{/each}

		</select>
	</div>
	<div class="break"></div>
	<div class="graphs">
		<div>
			<canvas id="white-graph" bind:this={whiteGraph}>
			</canvas>
		</div>
		<br/>
		<div>
			<canvas id="black-graph" bind:this={blackGraph}>
			</canvas>
		</div>
	</div>

	<div id="board" bind:this={board} style="width: 400px;"></div>
</main>


<style>
	/* graph-panel {
		width: 650px;
		display: flex;
		flex-direction: column;
		margin-right: 50px;
	} */
	.graphs {
		width: 650px;
		margin-left: auto;
	}
	main {
		display: flex;
		flex-wrap: wrap;
	}
	.row {
		margin-left: auto;
		margin-right: auto;
	}
	#board {
		margin-top: auto;
		margin-bottom: auto;
		margin-left: 40px;
		margin-right: auto;
	}
	.break {
		flex-basis: 100%;
		height: 0;
	}
	.graphs-and-board {
		display: flex;
		flex-direction: row;
	}
</style>